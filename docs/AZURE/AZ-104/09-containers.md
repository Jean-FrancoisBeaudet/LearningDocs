# Azure Containers (ACR, ACI, Container Apps)

> **Exam mapping:** AZ-104 --> "Deploy and manage Azure compute resources" (20-25%)
> **One-liner:** Build and store images in Azure Container Registry, run serverless containers with Azure Container Instances, and deploy scalable microservices on Azure Container Apps -- then size and scale each service appropriately.
> **Related:** [07-arm-and-bicep](./07-arm-and-bicep.md) | [08-virtual-machines](./08-virtual-machines.md) | [10-app-service](./10-app-service.md)

---

## 1. Azure Container Registry (ACR)

Azure Container Registry is a **managed, private Docker registry** service for storing and managing container images, Helm charts, and OCI artifacts. It integrates with Azure container orchestration services (ACI, ACA, AKS) and CI/CD pipelines.

**Reference:** [Introduction to Azure Container Registry](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-intro)

### 1.1 SKU Comparison

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| **Included storage** | 10 GiB | 100 GiB | 500 GiB |
| **Max storage** | 20 GiB | 200 GiB | 500 GiB per region |
| **Image throughput (ReadOps/min)** | 1,000 | 3,000 | 10,000 |
| **Webhooks** | 2 | 10 | 500 |
| **Geo-replication** | No | No | **Yes** |
| **Private link / private endpoints** | No | No | **Yes** (up to 200) |
| **Content trust (image signing)** | No | No | **Yes** |
| **Customer-managed keys** | No | No | **Yes** |
| **Repository-scoped tokens** | Yes | Yes | Yes |
| **Availability zones** | No | No | **Yes** |

**Reference:** [ACR service tiers and features](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-skus)

> **Exam trap:** Geo-replication and private link are **Premium-only** features. If a question asks about pulling images close to deployments in multiple regions, the answer requires Premium SKU.

### 1.2 Creating a Registry

```bash
# Create a resource group
az group create --name myRG --location eastus

# Create a Basic registry (name must be globally unique, alphanumeric only)
az acr create --resource-group myRG --name myregistry --sku Basic

# Upgrade to Premium later
az acr update --name myregistry --sku Premium
```

### 1.3 Pushing and Pulling Images

```bash
# Authenticate Docker client to ACR
az acr login --name myregistry

# Tag a local image for ACR
docker tag myapp:latest myregistry.azurecr.io/myapp:v1

# Push to ACR
docker push myregistry.azurecr.io/myapp:v1

# Pull from ACR
docker pull myregistry.azurecr.io/myapp:v1
```

The login token is valid for **3 hours**; re-run `az acr login` to refresh.

### 1.4 ACR Tasks (Quick Builds)

ACR Tasks lets you build images **in the cloud** without a local Docker daemon. Useful for CI/CD and automated builds.

```bash
# Quick build: send Dockerfile context to ACR and build remotely
az acr build --registry myregistry --image myapp:v1 .

# Automatically trigger builds on Git commit or base-image update
az acr task create --name buildOnCommit \
  --registry myregistry \
  --image myapp:{{.Run.ID}} \
  --context https://github.com/org/repo.git \
  --file Dockerfile \
  --git-access-token $PAT
```

**Reference:** [ACR Tasks overview](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tasks-overview)

### 1.5 Authentication Methods

| Method | Use Case | Notes |
|--------|----------|-------|
| **Individual Microsoft Entra ID** | Developer access, interactive | `az acr login` exchanges Entra token |
| **Service principal** | Unattended CI/CD, headless apps | Scoped roles (AcrPull, AcrPush, AcrDelete); create per-app principals |
| **Managed identity** | Azure service-to-ACR (ACI, ACA, AKS, App Service) | System-assigned or user-assigned; no credential management |
| **Repository-scoped token** | Granular access to specific repos | Token + scope map; useful for IoT devices or external consumers |
| **Admin account** | Quick testing only | Single shared username/password per registry; **not for production** |

**Reference:** [ACR authentication overview](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-authentication)

> **Exam trap:** The admin account is disabled by default and should **never** be used for production. Exam questions often test whether you know to use a service principal or managed identity instead.

### 1.6 Geo-Replication (Premium Only)

Geo-replication places registry replicas in multiple Azure regions, providing:

- **Network-close image pulls** -- containers pull from the closest replica.
- **Single management plane** -- one registry name, one push endpoint; replication is automatic.
- **Regional resilience** -- if one region goes down, images are still available from other replicas.

```bash
# Add a replica in West Europe
az acr replication create --registry myregistry --location westeurope

# List all replicas
az acr replication list --registry myregistry -o table
```

Storage is counted **per replica** -- a 1 GiB image replicated to 3 regions consumes 3 GiB of total storage.

**Reference:** [Geo-replication in ACR](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-geo-replication)

---

## 2. Azure Container Instances (ACI)

Azure Container Instances is a **serverless** service that runs containers **without managing VMs or orchestrators**. It is the fastest way to run an isolated container in Azure -- ideal for burst workloads, batch jobs, and simple applications.

**Reference:** [Azure Container Instances overview](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-overview)

### 2.1 Creating a Container Instance

```bash
# Run a public image with a public IP
az container create \
  --resource-group myRG \
  --name mycontainer \
  --image mcr.microsoft.com/azuredocs/aci-helloworld \
  --ports 80 \
  --dns-name-label my-aci-demo \
  --location eastus

# Run from a private ACR (using service principal or managed identity)
az container create \
  --resource-group myRG \
  --name mycontainer \
  --image myregistry.azurecr.io/myapp:v1 \
  --registry-login-server myregistry.azurecr.io \
  --registry-username $SP_APP_ID \
  --registry-password $SP_PASSWORD \
  --ports 80
```

### 2.2 Container Groups

A **container group** is a collection of containers scheduled on the **same host machine**. Containers in a group share:

- **Lifecycle** -- started, stopped, and deleted together.
- **Network** -- same IP address, same port space (localhost communication).
- **Storage volumes** -- can mount shared Azure Files volumes or emptyDir volumes.

| Deployment method | Multi-container support |
|-------------------|------------------------|
| **ARM template / Bicep** | Yes -- full control over all properties |
| **YAML file** (`az container create --file`) | Yes -- simpler syntax, recommended for ACI-only deployments |
| **CLI flags** | Single container only |

> **Exam trap:** Multi-container groups are currently supported **only on Linux** containers. Windows container instances are deployed as single-container instances.

**Reference:** [Container groups in ACI](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-container-groups)

### 2.3 Resource Allocation

Each container in a group specifies CPU and memory **requests**. The container group's total resources equal the sum of individual container requests.

| Resource | Linux max (per group) | Windows max (per group) |
|----------|-----------------------|-------------------------|
| **CPU cores** | 4 vCPU | 4 vCPU |
| **Memory** | 16 GB | 16 GB |
| **GPU** | Supported (NVIDIA Tesla, select regions) | Not supported |
| **Max image size** | 15 GB | 15 GB |

> Resource limits vary by region. Check [ACI resource and quota limits](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-resource-and-quota-limits) for current availability.

### 2.4 Restart Policies

| Policy | Behavior | Use Case |
|--------|----------|----------|
| **Always** (default) | Restart containers when they exit | Long-running services (web servers) |
| **OnFailure** | Restart only on non-zero exit code | Batch/ETL jobs that may fail |
| **Never** | Run once and stop | One-time tasks, scripts |

```bash
az container create \
  --resource-group myRG \
  --name batchjob \
  --image myregistry.azurecr.io/etl:v1 \
  --restart-policy OnFailure
```

### 2.5 Environment Variables and Secure Values

```bash
az container create \
  --resource-group myRG \
  --name mycontainer \
  --image myapp:v1 \
  --environment-variables 'APP_ENV=production' 'LOG_LEVEL=info' \
  --secure-environment-variables 'DB_PASSWORD=s3cret!'
```

- **`--environment-variables`** -- visible in container properties and logs.
- **`--secure-environment-variables`** -- encrypted at rest, not visible in portal or API responses.

### 2.6 Mounting Azure Files Volumes

```bash
az container create \
  --resource-group myRG \
  --name mycontainer \
  --image myapp:v1 \
  --azure-file-volume-share-name myshare \
  --azure-file-volume-account-name mystorageaccount \
  --azure-file-volume-account-key $STORAGE_KEY \
  --azure-file-volume-mount-path /mnt/data
```

Azure Files is the **only** durable storage option for ACI. EmptyDir volumes are available for temp storage within a container group but are lost when the group stops.

### 2.7 Networking Options

| Option | Details |
|--------|---------|
| **Public IP** | Assigned automatically with `--ip-address public`; combine with `--dns-name-label` for FQDN |
| **VNet deployment** | Deploy into an Azure virtual network subnet using `--vnet` and `--subnet` flags; no public IP, accessible only within the VNet |
| **No public IP** | Omit `--ip-address` to create a container with no external access |

**Reference:** [VNet scenarios for ACI](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-virtual-network-concepts)

### 2.8 Logging and Diagnostics

```bash
# View container logs (stdout/stderr)
az container logs --resource-group myRG --name mycontainer

# Attach to a running container's output stream
az container attach --resource-group myRG --name mycontainer

# Execute a command inside the container
az container exec --resource-group myRG --name mycontainer --exec-command "/bin/sh"

# Show container details (state, events, IP, etc.)
az container show --resource-group myRG --name mycontainer -o table
```

---

## 3. Azure Container Apps (ACA)

Azure Container Apps is a **serverless container platform** built on top of Kubernetes (powered by AKS under the hood) and [KEDA](https://keda.sh/) for event-driven scaling. It abstracts cluster management while providing microservices features like revisions, traffic splitting, and Dapr integration.

**Reference:** [Azure Container Apps overview](https://learn.microsoft.com/en-us/azure/container-apps/overview)

### 3.1 Key Concepts

| Concept | Description |
|---------|-------------|
| **Environment** | A secure boundary around one or more container apps. Apps in the same environment share the same virtual network, logging destination (Log Analytics), and Dapr configuration. |
| **Container App** | A single deployable unit; can contain one or more containers (sidecar pattern). |
| **Revision** | An immutable snapshot of a container app version. New revisions are created when you change the app's template (image, env vars, scale rules). |
| **Replica** | A running instance of a revision. Scaling adds/removes replicas. |

### 3.2 Creating a Container App

```bash
# Create an environment
az containerapp env create \
  --name myEnv \
  --resource-group myRG \
  --location eastus

# Create a container app with external ingress
az containerapp create \
  --name myapp \
  --resource-group myRG \
  --environment myEnv \
  --image myregistry.azurecr.io/myapp:v1 \
  --registry-server myregistry.azurecr.io \
  --registry-identity system \
  --target-port 8080 \
  --ingress external \
  --min-replicas 0 \
  --max-replicas 10
```

### 3.3 Revisions

| Mode | Behavior | Use Case |
|------|----------|----------|
| **Single revision** (default) | Old revision is deactivated when a new one is created | Standard rolling deployments |
| **Multiple revision** | Multiple revisions active simultaneously; traffic split by weight | Blue/green deployments, A/B testing, canary releases |

```bash
# Enable multiple revision mode
az containerapp revision set-mode \
  --name myapp \
  --resource-group myRG \
  --mode multiple

# Split traffic: 80% to current, 20% to new revision
az containerapp ingress traffic set \
  --name myapp \
  --resource-group myRG \
  --revision-weight myapp--rev1=80 myapp--rev2=20
```

**Reference:** [Revisions in Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/revisions)

### 3.4 Ingress

| Type | Visibility | URL Pattern |
|------|-----------|-------------|
| **External** | Accessible from the internet | `https://<app-name>.<env-unique-id>.<region>.azurecontainerapps.io` |
| **Internal** | Accessible only within the environment's VNet | Same FQDN pattern, but not publicly routable |
| **Disabled** | No HTTP endpoint | Background workers, event processors |

- Supports **HTTP** and **TCP** ingress.
- Built-in **TLS termination** -- no manual certificate required for the default domain.
- Custom domains with managed or BYO certificates supported.

**Reference:** [Configure ingress for Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/ingress-how-to)

### 3.5 Dapr Integration (Brief)

[Dapr](https://learn.microsoft.com/en-us/azure/container-apps/dapr-overview) (Distributed Application Runtime) runs as a sidecar alongside your container app and provides:

- **Service invocation** with mutual TLS and automatic retries.
- **State management** (Azure Cosmos DB, Redis, etc.).
- **Pub/sub messaging** (Azure Service Bus, Event Hubs, Kafka).
- **Bindings** (input/output to external systems).

Dapr is enabled per app:

```bash
az containerapp dapr enable \
  --name myapp \
  --resource-group myRG \
  --dapr-app-id myapp \
  --dapr-app-port 8080
```

> Dapr-enabled apps can invoke each other by app ID even **without** HTTP ingress enabled.

### 3.6 KEDA-Based Scaling

Azure Container Apps uses [KEDA](https://keda.sh/) under the hood to drive autoscaling. You define **scale rules** that map event sources to replica counts.

Built-in scale rule types:

| Rule Type | Trigger Source | Scale to Zero? |
|-----------|---------------|----------------|
| **HTTP** | Concurrent HTTP requests | Yes |
| **TCP** | Concurrent TCP connections | Yes |
| **Azure Queue Storage** | Queue message count | Yes |
| **Azure Service Bus** | Message count on topic/queue | Yes |
| **Custom (KEDA scaler)** | Any KEDA-supported scaler (Kafka, Redis, Cron, PostgreSQL, etc.) | Yes |
| **CPU** | CPU utilization % | **No** |
| **Memory** | Memory utilization % | **No** |

> **Exam trap:** CPU and memory scale rules **cannot** scale to zero because they require a running container to measure metrics.

---

## 4. Sizing and Scaling

### 4.1 ACI Sizing

ACI uses **fixed resource allocation** -- you specify CPU and memory at deployment time and they do not autoscale.

```bash
az container create \
  --resource-group myRG \
  --name mycontainer \
  --image myapp:v1 \
  --cpu 2 \
  --memory 4
```

| Constraint | Value |
|------------|-------|
| Max CPU per container group | 4 vCPU |
| Max memory per container group | 16 GB |
| GPU support | Linux only (NVIDIA Tesla, select regions/SKUs) |
| Autoscaling | **Not supported** -- resources are fixed at deployment |
| Scale-to-zero | Not applicable -- you stop/delete the container group |

> **Exam trap:** ACI does **not** autoscale. If a question requires automatic horizontal scaling, the answer is ACA (or AKS), not ACI.

### 4.2 ACA Scaling

ACA provides **event-driven autoscaling** powered by KEDA with configurable min/max replica counts.

```bash
# Add an HTTP scale rule
az containerapp update \
  --name myapp \
  --resource-group myRG \
  --min-replicas 0 \
  --max-replicas 20 \
  --scale-rule-name http-rule \
  --scale-rule-type http \
  --scale-rule-http-concurrency 50

# Add a queue-based scale rule
az containerapp update \
  --name myapp \
  --resource-group myRG \
  --scale-rule-name queue-rule \
  --scale-rule-type azure-queue \
  --scale-rule-metadata "queueName=myqueue" "queueLength=10" "accountName=mystorage" \
  --scale-rule-auth "connection=connection-string-secret"
```

**Scaling behavior parameters:**

| Parameter | Description | Default |
|-----------|-------------|---------|
| **Polling interval** | How often KEDA queries event sources | 30 seconds |
| **Cool-down period** | Wait time after last event before scaling down to min replicas | 300 seconds |
| **Scale-up step** | Replicas added per scale-out: starts at 1, doubles (1 -> 4 -> 8 -> 16 -> 32...) | Progressive |
| **Scale-down step** | Replicas removed per scale-in | 100% of excess |

**Reference:** [Set scaling rules in Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/scale-app)

> **Exam trap:** ACA can **scale to zero** (0 replicas) with HTTP, TCP, and event-based rules -- meaning you pay nothing when idle. But CPU/memory rules cannot scale to zero.

### 4.3 ACI vs ACA Comparison

| Dimension | ACI | ACA |
|-----------|-----|-----|
| **Model** | Serverless single container / container group | Serverless container platform (Kubernetes-based) |
| **Autoscaling** | No -- fixed CPU/memory | Yes -- KEDA event-driven, min/max replicas |
| **Scale to zero** | No (stop/delete to stop billing) | Yes (HTTP, TCP, event rules) |
| **Multi-container** | Container groups (Linux only) | Sidecar containers per app |
| **Networking** | Public IP or VNet injection | Environment VNet, external/internal ingress |
| **Revisions / traffic splitting** | No | Yes -- blue/green, canary |
| **Dapr** | No | Built-in sidecar |
| **Built-in load balancing** | No | Yes (per-revision) |
| **GPU support** | Yes (Linux, select regions) | Limited |
| **Restart policies** | Always, OnFailure, Never | Managed by platform |
| **Best for** | Burst compute, batch jobs, simple single-container workloads, CI/CD runners | Microservices, web APIs, event-driven apps, long-running services needing autoscale |
| **Billing** | Per-second (vCPU + memory + optional GPU) | Per-second per active replica (scale-to-zero = no cost) |

**Decision rule of thumb:**
- Need to run a quick container with no orchestration? --> **ACI**
- Need autoscaling, revisions, traffic splitting, or Dapr? --> **ACA**
- Need full Kubernetes control (custom operators, node pools)? --> **AKS** (outside AZ-104 scope)

---

## Exam Traps

1. **ACR Premium for geo-replication and private link** -- Both features require the Premium SKU. Standard and Basic do not support them. If a scenario involves multi-region image distribution or private network access to the registry, Premium is the only valid answer.

2. **ACI does not autoscale** -- ACI provides fixed CPU/memory allocation. There is no built-in autoscaler. If horizontal scaling is needed, use ACA or AKS.

3. **ACA scale-to-zero** -- ACA can scale to 0 replicas with HTTP, TCP, and event-based (queue, Service Bus) rules, meaning zero cost when idle. However, CPU/memory-based scale rules **cannot** scale to zero because there must be a running container to measure utilization.

4. **ACI container groups are Linux-only for multi-container** -- Windows containers on ACI can only run as single-container instances. Multi-container groups (sidecar pattern, init containers) require Linux.

5. **ACR admin account is not for production** -- The admin account provides a single shared credential. Exam answers involving production workloads should use service principals, managed identities, or repository-scoped tokens. The admin account is acceptable only for quick dev/test scenarios.

6. **ACR login token expiry** -- The `az acr login` token expires after **3 hours**. Questions about failed pushes after a period of time may hinge on token refresh.

7. **ACA revisions are immutable** -- Changing environment variables, images, or scale rules creates a **new revision**. You cannot modify an existing revision in place.

8. **ACI secure environment variables** -- Use `--secure-environment-variables` for secrets. Regular environment variables are visible in the portal and API. Expect scenario questions testing whether a connection string is properly secured.

---

## Sources

- [Introduction to Azure Container Registry](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-intro)
- [ACR service tiers and features (SKUs)](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-skus)
- [ACR authentication overview](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-authentication)
- [Service principal authentication for ACR](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-auth-service-principal)
- [Managed identity authentication for ACR](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-authentication-managed-identity)
- [Geo-replication in ACR](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-geo-replication)
- [ACR Tasks overview](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tasks-overview)
- [Azure Container Instances overview](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-overview)
- [Container groups in ACI](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-container-groups)
- [ACI resource and quota limits](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-resource-and-quota-limits)
- [VNet scenarios for ACI](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-virtual-network-concepts)
- [Azure Container Apps overview](https://learn.microsoft.com/en-us/azure/container-apps/overview)
- [Revisions in Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/revisions)
- [Configure ingress for Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/ingress-how-to)
- [Dapr in Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/dapr-overview)
- [Set scaling rules in Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/scale-app)
- [Set up a private endpoint with private link for ACR](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-private-link)
