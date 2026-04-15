# Exam Cheatsheet

> **Exam mapping:** AZ-204 · AZ-305 · AI-102 *(→ AI-103)* · AI-300
> **One-liner:** The traps Microsoft exams love to set on Cosmos DB, with the "real answer" for each.
> **Related:** links out to every other note in this folder.

## The 20 things most likely to show up

### 1. Consistency level defaults
- **Session** is the default. [`04-consistency-levels.md`](04-consistency-levels.md)
- Strong is **not** compatible with multi-region writes.
- Per-request override can only **weaken** from the account default.

### 2. Throughput modes
- Serverless caps: **5 000 RU/s**, **1 TB** per container. [`03-throughput-and-ru.md`](03-throughput-and-ru.md)
- Autoscale premium: **1.5×** manual rate; scales **10 % → max**.
- Autoscale entry point: **1 000 RU/s** (so floor 100, ceiling 1 000).

### 3. Partition key rules
- 20 GB **logical partition** ceiling. [`02-partitioning.md`](02-partitioning.md)
- Hierarchical partition keys allow up to **3 levels**, each path bounds the other.
- Can't change partition key after creation — recreate + migrate.

### 4. API is immutable
- API picked at account creation stays forever. [`01-apis-and-data-models.md`](01-apis-and-data-models.md)

### 5. Vectors & AI
- DiskANN = best for **> 50 k vectors**; NoSQL + Mongo vCore. [`08-ai-and-vector-search.md`](08-ai-and-vector-search.md)
- NoSQL hybrid search uses **RRF**; no LLM-tuned semantic ranker (that's AI Search).
- Enable `EnableNoSQLVectorSearch` capability first.

### 6. SLA composition
- 99.999% read SLA needs **multi-region + SDK direct mode + preferred regions**. [`05-global-distribution-and-ha.md`](05-global-distribution-and-ha.md)
- 99.999% write SLA = **multi-region writes enabled**.

### 7. Auth
- Prefer **data-plane RBAC + Managed Identity**; `disableLocalAuth: true`. [`09-security.md`](09-security.md)
- Data-plane RBAC uses `az cosmosdb sql role assignment`, not the ARM `az role assignment`.

### 8. Change feed vs Event Grid vs Event Hubs
- Change feed = ordered per-partition stream of changes. [`07-change-feed-and-analytics.md`](07-change-feed-and-analytics.md)
- Event Grid = pub/sub fan-out (discrete events, not content).
- Event Hubs = high-volume ingestion, consumer groups.

### 9. Analytics / HTAP
- **Analytical store** (Synapse Link) must be enabled at **container creation** — not retro.
- **Fabric mirroring** is the newer, recommended HTAP path; no analytical store needed.

### 10. Backups
- Periodic (default, 2 copies) vs **Continuous 7/30 days** (PITR).
- Restore is always **to a new account**. Can't overwrite in place.
- Continuous 30 days required for "all-versions-and-deletes" change feed.

### 11. Query cost traps
- `OFFSET/LIMIT` costs same as the full scan — use continuation tokens. [`06-indexing-and-query.md`](06-indexing-and-query.md)
- `ORDER BY` without composite index → in-memory sort, high RU.
- Point-read `ReadItemAsync(id, pk)` = **1 RU**, always better than SELECT.

### 12. Global writes with conflicts
- Default LWW on `_ts`; can use custom property or stored-proc merge.
- Unresolved conflicts → **conflict feed** you can drain.

### 13. Cosmos vs Azure SQL vs AI Search
- Cosmos: distributed JSON + vector, RU model.
- Azure SQL: relational, T-SQL, vector preview on PaaS.
- AI Search: search engine with **LLM-powered semantic ranker**, better for enterprise search.

### 14. Zone redundancy
- Optional, ~1.25× cost; needed for highest in-region availability.

### 15. Network hardening
- Private Endpoint + `publicNetworkAccess: Disabled`.
- One private endpoint **per region** that needs local access.

### 16. Encryption
- At rest: always; CMK optional with Key Vault + Managed Identity.
- **Always Encrypted** = client-side encryption of specific properties.

### 17. Functions Cosmos trigger
- Requires **leases** container; never share leases across functions.
- Extension **v4.7+** supports Managed Identity via `__accountEndpoint` / `__credential` settings.

### 18. SDK singleton + direct mode
- `CosmosClient` is thread-safe. Instantiate **once**; Direct/TCP mode on.

### 19. TTL
- Free deletes; set per container or per item. Perfect for LLM cache, session, logs.

### 20. Serverless ↔ Provisioned migration
- **GA both directions**, in-place, no downtime. [`03-throughput-and-ru.md`](03-throughput-and-ru.md)

## Scenario drill patterns

**"Global app, active-active, strong ordering, lowest latency."**
→ Multi-region writes + **Bounded Staleness** + Direct mode + preferred regions + zone redundancy. *Strong is a trap here.*

**"Chatty queries are hot-partitioning on `tenantId`."**
→ Hierarchical partition keys `(tenantId, userId)` or synthetic key. Not "add more RU/s".

**"429s despite raising RU/s to 50 000."**
→ Hot logical partition. Check partition-key RU/s chart; redesign key.

**"Build a RAG app with freshest possible data, operational DB already Cosmos."**
→ Keep embeddings in Cosmos NoSQL + DiskANN. Don't add AI Search unless you need semantic ranker.

**"Archive docs older than 30 days, cheap."**
→ `ttl` + optional Fabric mirroring for long-term analytics copy. Not "custom Function to delete".

**"Audit trail of every change, including deletes."**
→ Change feed **all-versions-and-deletes** mode + continuous backup on. Not regular change feed.

**"Mobile app, occasional traffic spikes, tiny overall load."**
→ Serverless, Session consistency, partition by userId.

**"Secrets in app settings bad, but SDK needs account key."**
→ Data-plane RBAC + `DefaultAzureCredential`; disable local auth after rollout.

## Final study tips

- Memorize the **consistency level table** (RU cost, SLA, trade-offs) cold.
- Memorize **autoscale 10 % / 1.5× / 1 000 RU entry**.
- Memorize **20 GB logical partition limit**.
- Be able to read a Bicep/ARM Cosmos resource and spot misconfigurations (e.g. `disableLocalAuth: false`, default consistency wrong, single-region when multi-region is required).
- Know which features are **account-level vs container-level vs item-level** — Microsoft questions often hide the test in this distinction.

## Sources

All linked from the individual topic notes in this folder. Start with the [Cosmos DB learning paths on Microsoft Learn](https://learn.microsoft.com/en-us/training/browse/?products=azure-cosmos-db) and the [AZ-204 study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-204) / [AZ-305 study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-305).
