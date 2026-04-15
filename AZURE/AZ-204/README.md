# AZ-204 — Developing Solutions for Microsoft Azure

**Advanced overview and study roadmap.** Based on the official Microsoft Learn *Skills measured as of January 14, 2026* ([study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-204)).

> ⚠️ **Exam retires July 31, 2026 (11:59 PM CST).** If you're starting now (April 2026), you have a tight window. Schedule early and be aware a successor/renewal path may emerge — check Microsoft Learn before booking.

---

## Exam at a glance

| Item | Detail |
|---|---|
| Exam code | AZ-204 |
| Role | Azure Developer Associate |
| Questions | 40–60 (MCQ, drag-drop, case studies, code snippets, active-screen) |
| Duration | 120 min (+30 min if English isn't your first language) |
| Passing score | 700 / 1000 (scaled — not 70%) |
| Languages | English, Japanese, Chinese (Simplified), Korean, German, French, Spanish, Portuguese (Brazil), Arabic (Saudi Arabia), Russian, Chinese (Traditional), Italian, Indonesian (Indonesia) |
| Prereqs (recommended) | 2+ yrs dev experience, C# or Python proficiency, Azure CLI/PowerShell fluency, REST/JSON |
| Retirement | **July 31, 2026** |

### Skill domains & weights

| # | Domain | Weight |
|---|---|---|
| 1 | Develop Azure compute solutions | **25–30%** |
| 2 | Develop for Azure storage | **15–20%** |
| 3 | Implement Azure security | **15–20%** |
| 4 | Monitor, troubleshoot, and optimize Azure solutions | **5–10%** |
| 5 | Connect to and consume Azure services and third-party services | **20–25%** |

---

## How to use this roadmap

Each section below gives you:

1. **What it covers** — the official sub-objectives.
2. **Key services & concepts** — what to master at architect/senior-dev depth.
3. **Exam traps** — the distinctions MS loves to test.
4. **Deep-dive prompt** — copy/paste into a future `claude /azure-expert` session to generate a dedicated study note.

Deep-dive notes should be saved as separate files alongside this one (e.g. `AZURE/AZ-204/01-compute-containers.md`).

---

## 1. Develop Azure compute solutions (25–30%)

### 1a. Implement containerized solutions

**Covers:** Building/managing container images, Azure Container Registry (ACR), Azure Container Instances (ACI), Azure Container Apps (ACA).

**Key concepts:** Dockerfile best practices, multi-stage builds, ACR tasks & geo-replication, ACR authentication (admin user vs managed identity vs token), ACI sidecars & container groups, ACA revisions/traffic splitting/Dapr/KEDA-based scaling, ingress modes, secrets.

**Exam traps:**
- ACI vs ACA vs AKS — when to pick each (ACA is the sweet spot for microservices without K8s overhead).
- ACR tasks (build/run/multi-step) vs pulling from Docker Hub.
- ACA scale-to-zero behavior and cold-start implications.

**Deep-dive prompt:**
> `/azure-expert` Create `AZURE/AZ-204/01a-containers.md` — a deep-dive on AZ-204 containerized solutions covering Dockerfile authoring, ACR (SKUs, tasks, geo-replication, MI auth, content trust), ACI (container groups, YAML, volumes, restart policies), and Azure Container Apps (environments, revisions, ingress, Dapr, KEDA scale rules, secrets, managed identity). Include Bicep + az CLI snippets, ACI vs ACA vs AKS decision table, and exam traps.

---

### 1b. Implement Azure App Service Web Apps

**Covers:** Creating web apps, diagnostics/logging, code & container deployment, TLS/API/connection config, autoscaling, deployment slots.

**Key concepts:** App Service plans (F/D/B/S/P v3/Isolated v2), OS + runtime stacks, deployment methods (ZIP, Run From Package, CI/CD, container), slot swap & sticky settings, autoscale rules (metric/schedule), health checks, App Service Auth ("Easy Auth"), hybrid connections, VNet integration vs Private Endpoint.

**Exam traps:**
- Which settings are **slot-specific** (sticky) vs swappable.
- Autoscale requires Standard+; Always On availability by tier.
- Premium v3 vs Isolated v2 — when network isolation requires ASE.
- `WEBSITE_RUN_FROM_PACKAGE` behavior.

**Deep-dive prompt:**
> `/azure-expert` Create `AZURE/AZ-204/01b-app-service.md` — a deep-dive on AZ-204 App Service covering plan SKUs/tiers (incl. P v3, Isolated v2), deployment options (ZIP, Run From Package, containers, GitHub Actions, DevOps), diagnostics (App Service logs, Log Stream, Kudu, App Insights), TLS/custom domains/SNI vs IP SSL, autoscale rules, deployment slots (swap mechanics, sticky settings, warmup), networking (VNet integration, Private Endpoint, hybrid connections), and Easy Auth. Include Bicep + az CLI + C# snippets and exam traps.

---

### 1c. Implement Azure Functions

**Covers:** Function apps, input/output bindings, triggers (data ops, timers, webhooks).

**Key concepts:** Hosting plans (Consumption, Flex Consumption, Premium, Dedicated), isolated worker vs in-process (.NET 8+ is **isolated only**), durable functions patterns (chaining, fan-out/fan-in, async HTTP, monitor, aggregator, human interaction), bindings configuration (`function.json` vs attributes), cold starts, concurrency, retries, filters, Durable Entities.

**Exam traps:**
- Consumption timeout = 5 min default / 10 min max vs Premium 30+ min.
- Durable Function patterns and which to pick.
- Binding directions (`in`/`out`/`inout`) and trigger = one per function.
- Cold start mitigation (Premium pre-warmed, Flex Consumption always-ready).

**Deep-dive prompt:**
> `/azure-expert` Create `AZURE/AZ-204/01c-functions.md` — a deep-dive on AZ-204 Azure Functions covering hosting plans (Consumption, Flex Consumption, Premium EP, Dedicated), isolated worker model, triggers + input/output bindings (HTTP, Timer, Blob, Queue, Service Bus, Event Grid, Event Hub, Cosmos DB), Durable Functions patterns (chaining, fan-out/fan-in, async HTTP, monitor, aggregator, human interaction) and Durable Entities, retries/filters, scaling/concurrency, host.json settings, and deployment. Include C# isolated-worker snippets, host.json samples, and exam traps.

---

## 2. Develop for Azure storage (15–20%)

### 2a. Azure Cosmos DB

**Covers:** SDK operations on containers/items, consistency levels, change feed.

**Key concepts:** API choice (NoSQL, MongoDB, Cassandra, Gremlin, Table, PostgreSQL) — **exam = NoSQL (Core SQL) API**, request units (RUs), partitioning (logical vs physical), partition-key design, 5 consistency levels (Strong, Bounded Staleness, Session, Consistent Prefix, Eventual), change feed (push via Functions trigger, pull via SDK), TTL, indexing policy, server-side programming (stored procs/triggers/UDFs), multi-region writes, serverless vs provisioned vs autoscale throughput.

**Exam traps:**
- Consistency level trade-offs (latency vs RU cost vs guarantees). Session is the default.
- Cross-partition queries cost more RUs.
- Change feed does **not** capture deletes (use soft-delete + TTL).
- Optimistic concurrency with ETag / `IfMatch`.

**Deep-dive prompt:**
> `/azure-expert` Create `AZURE/AZ-204/02a-cosmos-db.md` — a deep-dive on AZ-204 Cosmos DB (NoSQL API) covering the 5 consistency levels with trade-offs, partition-key design, RU economics (provisioned/autoscale/serverless), SDK CRUD + query patterns in C#, change feed (push via Functions, pull model), TTL, indexing policy tuning, optimistic concurrency (ETag), multi-region writes, and conflict resolution. Include code snippets, decision tables, and exam traps.

---

### 2b. Azure Blob Storage

**Covers:** Properties/metadata, SDK data operations, storage policies & lifecycle management.

**Key concepts:** Account kinds (StorageV2, BlockBlobStorage), redundancy (LRS/ZRS/GRS/RA-GRS/GZRS/RA-GZRS), access tiers (Hot/Cool/Cold/Archive) + rehydration, blob types (block/append/page), metadata vs system properties, lifecycle management rules, immutable (WORM) policies, soft delete + versioning, SAS (user-delegation vs account vs service), copy operations (sync/async, Put Blob From URL), AzCopy.

**Exam traps:**
- Archive rehydration priorities (Standard ≤15h / High ≤1h) and cost.
- User delegation SAS (Entra-backed) vs account SAS — prefer user delegation.
- Lifecycle rules run once/day, not real-time.
- Changing tier cost implications (early deletion fees for Cool/Cold/Archive).

**Deep-dive prompt:**
> `/azure-expert` Create `AZURE/AZ-204/02b-blob-storage.md` — a deep-dive on AZ-204 Azure Blob Storage covering account kinds, redundancy options, access tiers + rehydration, blob types, properties/metadata (and case sensitivity), SDK operations in C# (upload, download, copy, lease, stream), lifecycle management rules, immutable policies, soft delete + versioning, SAS types (user-delegation, account, service) + stored access policies. Include Bicep + az CLI + C# snippets, lifecycle JSON, and exam traps.

---

## 3. Implement Azure security (15–20%)

### 3a. User authentication and authorization

**Covers:** Microsoft Identity Platform, Microsoft Entra ID, SAS, Microsoft Graph.

**Key concepts:** OAuth 2.0 / OIDC flows (auth code + PKCE, client credentials, on-behalf-of, device code), MSAL (C#, JS), app registrations vs enterprise apps, delegated vs application permissions, admin consent, token caching, scopes vs roles, SAS types (user delegation is preferred), Microsoft Graph SDK patterns, least privilege.

**Exam traps:**
- Pick the right OAuth flow for each scenario (SPA = auth code + PKCE; daemon = client credentials; web API calling downstream = on-behalf-of).
- Delegated vs application permissions in Graph.
- User-delegation SAS requires Entra identity + RBAC on storage.

**Deep-dive prompt:**
> `/azure-expert` Create `AZURE/AZ-204/03a-identity-auth.md` — a deep-dive on AZ-204 authN/authZ covering Microsoft Identity Platform, OAuth 2.0/OIDC flows (auth code + PKCE, client credentials, on-behalf-of, device code, ROPC — and when each applies), MSAL.NET samples, app registration vs enterprise app, delegated vs application permissions, admin consent, token lifetimes/caching, Microsoft Graph SDK patterns, and SAS (user-delegation, account, service). Include C# snippets and exam traps.

---

### 3b. Secure Azure solutions (Key Vault, App Config, Managed Identity)

**Covers:** App Configuration / Key Vault for config, Key Vault SDK for keys/secrets/certs, Managed Identities.

**Key concepts:** Key Vault access models (RBAC — preferred, vs access policies), Key Vault references in App Service/Functions, App Configuration (feature flags, labels, dynamic refresh, Key Vault refs), system-assigned vs user-assigned managed identity, `DefaultAzureCredential` chain, rotation strategies, soft-delete + purge protection.

**Exam traps:**
- RBAC model vs vault access policies — RBAC is now the default and MS-recommended.
- MI cannot be used outside Azure-hosted resources (use `DefaultAzureCredential` for local dev).
- Key Vault references auto-resolve at App Service runtime only if MI is assigned and granted `get` on secret.
- App Configuration labels + sentinel key for dynamic refresh.

**Deep-dive prompt:**
> `/azure-expert` Create `AZURE/AZ-204/03b-keyvault-appconfig-mi.md` — a deep-dive on AZ-204 Azure Key Vault (keys/secrets/certs, RBAC vs access policies, soft-delete + purge protection, rotation, Key Vault references in App Service/Functions), Azure App Configuration (feature flags, labels, sentinel key, dynamic refresh, Key Vault refs), and Managed Identities (system- vs user-assigned, `DefaultAzureCredential`, federated credentials). Include Bicep + C# snippets and exam traps.

---

## 4. Monitor, troubleshoot, and optimize Azure solutions (5–10%)

**Covers:** Azure Monitor **Application Insights** — metrics/logs/traces analysis, availability tests, alerts, app instrumentation.

**Key concepts:** App Insights resources (workspace-based is the standard), auto-instrumentation vs manual SDK, distributed tracing + correlation, dependencies, sampling (adaptive, fixed, ingestion), custom events/metrics/traces, KQL on `requests`/`dependencies`/`traces`/`exceptions`/`customEvents`, availability tests (URL ping — being deprecated — and standard tests), alert rules (metric, log, activity log), action groups, Live Metrics Stream.

**Exam traps:**
- Workspace-based vs classic App Insights (classic is retired).
- Sampling strategy impact on counts + cost.
- Availability test types and retirement timeline (URL ping tests).
- Correlation IDs (`operation_Id`, `operation_ParentId`) across services.

**Deep-dive prompt:**
> `/azure-expert` Create `AZURE/AZ-204/04-monitoring-appinsights.md` — a deep-dive on AZ-204 monitoring with Azure Monitor Application Insights covering workspace-based resources, instrumentation (auto vs SDK), distributed tracing + correlation IDs, dependency tracking, sampling (adaptive/fixed/ingestion), custom events/metrics/traces, KQL queries across `requests`/`dependencies`/`traces`/`exceptions`, availability tests (standard tests, retirement of URL ping), alert rules + action groups, Live Metrics. Include C# instrumentation snippets, KQL queries, and exam traps.

---

## 5. Connect to and consume Azure services and third-party services (20–25%)

### 5a. Azure API Management (APIM)

**Covers:** Creating APIM, API design/docs, access control, policies.

**Key concepts:** Tiers (Consumption, Basic v2, Standard v2, Premium v2, Developer), products, subscriptions, subscription keys, OAuth 2.0 / JWT validation, policies (inbound/backend/outbound/on-error) — rate-limit, quota, caching, CORS, set-header, rewrite-uri, mock, send-request, validate-jwt, named values + Key Vault refs, versions vs revisions, developer portal.

**Exam traps:**
- Policy scopes (global → product → API → operation) and inheritance via `<base />`.
- Versioning strategies (path, header, query string) vs revisions (non-breaking).
- Consumption vs v2 tiers — VNet, caching, developer portal availability.

**Deep-dive prompt:**
> `/azure-expert` Create `AZURE/AZ-204/05a-api-management.md` — a deep-dive on AZ-204 Azure API Management covering tiers (Consumption, v2 tiers, Premium, Developer) and capability matrix, products/subscriptions/subscription keys, versions vs revisions, policy pipeline (inbound/backend/outbound/on-error) with `<base />` inheritance, key policies (rate-limit-by-key, quota, cache-lookup/store, validate-jwt, set-backend-service, rewrite-uri, mock-response, send-request), named values + Key Vault refs, OAuth/JWT auth, developer portal. Include policy XML snippets, Bicep, and exam traps.

---

### 5b. Event-based solutions (Event Grid, Event Hubs)

**Covers:** Solutions using Event Grid and Event Hubs.

**Key concepts:**
- **Event Grid** — discrete events, push model, CloudEvents schema vs Event Grid schema, topics vs system topics vs domains, event filtering, dead-lettering, retry policy, webhook validation handshake.
- **Event Hubs** — telemetry/stream ingest at scale, partitions, consumer groups, EventProcessorClient + checkpointing to Blob, Kafka endpoint, throughput units (standard) / processing units (premium) / capacity units (dedicated), Capture to Blob/ADLS.

**Exam traps:**
- Event Grid vs Event Hubs vs Service Bus — discrete event notification vs stream vs enterprise messaging.
- At-least-once delivery semantics for all three — design idempotent consumers.
- Event Hubs partition count is (mostly) immutable; choose carefully.
- Event Grid webhook endpoint validation handshake.

**Deep-dive prompt:**
> `/azure-expert` Create `AZURE/AZ-204/05b-event-grid-hubs.md` — a deep-dive on AZ-204 event-based solutions covering Azure Event Grid (schemas, topics vs system topics vs domains, subscriptions, filtering, retry + dead-letter, webhook validation, MQTT broker) and Azure Event Hubs (partitions, consumer groups, EventProcessorClient + checkpointing, Kafka surface, Capture, throughput/processing/capacity units, Geo-DR), with a comparison matrix of Event Grid vs Event Hubs vs Service Bus. Include C# producer/consumer snippets and exam traps.

---

### 5c. Message-based solutions (Service Bus, Queue Storage)

**Covers:** Solutions using Azure Service Bus and Azure Queue Storage.

**Key concepts:**
- **Service Bus** — queues and topics/subscriptions, sessions (FIFO + state), dead-letter queue, duplicate detection, scheduled messages, message deferral, transactions, peek-lock vs receive-delete, auto-forwarding, filters/rules (SQL, correlation, boolean), premium tier + geo-DR, JMS 2.0.
- **Queue Storage** — simple queues, visibility timeout, up to 64 KB messages (or 64 KB base64, ~48 KB effective), poison message handling with dequeue count.

**Exam traps:**
- **Service Bus vs Queue Storage** — when to pick which (size, features, ordering, transactions, cost).
- Peek-lock + `CompleteAsync`/`AbandonAsync`/`DeadLetterAsync` lifecycle.
- Sessions required for ordered processing and session state.
- Duplicate detection window + idempotent message IDs.

**Deep-dive prompt:**
> `/azure-expert` Create `AZURE/AZ-204/05c-service-bus-queue-storage.md` — a deep-dive on AZ-204 message-based solutions covering Azure Service Bus (queues, topics + subscriptions with SQL/correlation filters, sessions + FIFO + session state, peek-lock vs receive-delete, DLQ, duplicate detection, scheduled messages, deferral, transactions, auto-forwarding, premium tier + geo-DR) and Azure Queue Storage (limits, visibility timeout, poison messages). Include a Service Bus vs Queue Storage vs Event Grid vs Event Hubs decision matrix, C# snippets with `ServiceBusClient`/`QueueClient`, and exam traps.

---

## Suggested study sequence

1. **Week 1–2** — Compute (App Service → Functions → Containers). Biggest weight, and everything else plugs into compute.
2. **Week 3** — Storage (Blob → Cosmos DB).
3. **Week 4** — Security (Identity → Key Vault/App Config/MI). Cross-cutting — expect it in case studies.
4. **Week 5** — Integration (APIM → Messaging → Events).
5. **Week 6** — Monitoring (small weight but easy marks) + full practice assessments on Microsoft Learn.

## Hands-on labs (must-do)

- Deploy a container to ACA with revisions + traffic split + MI-authenticated ACR pull.
- Build a Durable Function fan-out/fan-in orchestration with Blob trigger.
- Implement a Cosmos DB change feed consumer via Azure Function.
- Wire App Service → Key Vault references with system-assigned MI.
- APIM policy: JWT validation + rate-limit-by-key + cache-lookup + Key Vault-backed named value.
- Service Bus topic + two subscriptions with SQL filters, session processing, DLQ handling.
- App Insights with distributed tracing across App Service → Service Bus → Function → Cosmos.

## Official resources

- [AZ-204 study guide (Microsoft Learn)](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-204)
- [AZ-204 exam page](https://learn.microsoft.com/en-us/credentials/certifications/exams/az-204/)
- [Free practice assessment](https://learn.microsoft.com/en-us/credentials/certifications/exams/az-204/practice/assessment?assessment-type=practice&assessmentId=35)
- [Exam Readiness Zone videos](https://learn.microsoft.com/en-us/shows/exam-readiness-zone/?terms=az-204)
- [Exam sandbox (UI demo)](https://aka.ms/examdemo)
