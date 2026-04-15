# AI Applications & Vector Search

> **Exam mapping:** AI-102 *(→ AI-103)* · AI-300 · AZ-204 *(integrated-vector SDK patterns)*
> **One-liner:** Cosmos DB is Microsoft's "unified AI database" — store operational data **and** embeddings in the same document, index with **DiskANN**, and run hybrid (vector + BM25 + semantic) search without a separate vector store.
> **Related:** [APIs & data models](01-apis-and-data-models.md) · [Indexing & query](06-indexing-and-query.md) · [Security](09-security.md)

## Why a single store for RAG

Separate vector DB (Pinecone, Qdrant, etc.) means:
- Two consistency models to reconcile (source doc updated, embedding not).
- Cross-system joins at query time.
- Extra security perimeter.

Cosmos DB keeps the **source document and its embedding in one item**, updated transactionally. Great for RAG, agent memory, LLM response caching.

## Where vector search is available

| API | Index types | Max dimensions | Notes |
|-----|-------------|----------------|-------|
| **NoSQL API** | `flat`, `quantizedFlat`, **`diskANN`** | **16 000** (with product quantization) | Hybrid search (vector + BM25 + semantic rerank) GA |
| **MongoDB vCore** (Azure DocumentDB) | `IVF`, **`HNSW`**, **`DiskANN` + filtered vector** | 2 000 typical, higher w/ PQ | Mongo-native API; GA |
| **PostgreSQL** (Citus) | `pgvector` (HNSW, IVF) + DiskANN extension | depends on pgvector | Regular Postgres tooling |
| **MongoDB RU** | limited IVF | lower | Not recommended for new AI apps |

### When to pick which

- New .NET / Python app, RAG heavy → **NoSQL API + DiskANN**.
- Existing Mongo app, need vector → **Mongo vCore + DiskANN**.
- Relational + vector, Postgres ecosystem → **Cosmos DB for PostgreSQL + pgvector/DiskANN**.

## NoSQL API: enabling vectors

Enable the capability on the account: `EnableNoSQLVectorSearch`.

Container definition (Bicep excerpt):

```jsonc
{
  "vectorEmbeddingPolicy": {
    "vectorEmbeddings": [
      {
        "path": "/embedding",
        "dataType": "float32",
        "distanceFunction": "cosine",
        "dimensions": 1536
      }
    ]
  },
  "indexingPolicy": {
    "vectorIndexes": [
      { "path": "/embedding", "type": "diskANN" }
    ],
    "excludedPaths": [ { "path": "/embedding/*" } ]   // don't also range-index the vector
  }
}
```

Query:

```sql
SELECT TOP 10 c.id, c.title,
  VectorDistance(c.embedding, @queryVector) AS score
FROM c
WHERE c.tenantId = @tenantId                           -- pre-filter by partition key
ORDER BY VectorDistance(c.embedding, @queryVector)
```

C# SDK call:

```csharp
var query = new QueryDefinition(sql)
    .WithParameter("@tenantId", tenantId)
    .WithParameter("@queryVector", embeddings);        // float[]
var iter = container.GetItemQueryIterator<Doc>(query);
```

### DiskANN characteristics

- State-of-the-art ANN algorithm from Microsoft Research — disk-based graph index.
- Scales to **billions of vectors** with low memory footprint.
- Microsoft cites **< 20 ms P99 query latency over 10 M vectors** and ~**43× lower cost vs Pinecone serverless** in recent benchmarks.
- Best when the candidate set is **> 50 k vectors**; for smaller sets, `quantizedFlat` or `flat` is cheaper.

### Index-type cheat sheet

| Index | When to use |
|-------|-------------|
| `flat` | < 10 k vectors; perfect recall; cheapest to maintain |
| `quantizedFlat` | 10 k – 100 k vectors; PQ reduces memory |
| `diskANN` | > 50 k vectors; best latency + cost at scale |

## Hybrid search (NoSQL API)

Combine vector + full-text + semantic rerank in one query using **Reciprocal Rank Fusion (RRF)**:

```sql
SELECT TOP 20 c.id, c.title
FROM c
WHERE FullTextContainsAny(c.body, "wireless", "noise cancelling")
ORDER BY RANK RRF(
  VectorDistance(c.embedding, @queryVector),
  FullTextScore(c.body, ["wireless", "noise cancelling"])
)
```

Pre-filter with partition key whenever possible — vector queries over a single logical partition are an order of magnitude cheaper than cross-partition fan-out.

## LLM response cache pattern

Cache expensive generations keyed by prompt embedding:

1. Compute embedding of user prompt with Azure OpenAI `text-embedding-3-small`.
2. `VectorDistance` query against a `llm_cache` container; if score > 0.95, return cached completion.
3. Otherwise call the LLM, store `{ prompt, embedding, completion, _ts }`.
4. Expire via TTL.

Typical savings: 30–60 % of token spend on repetitive assistants.

## Integrations worth knowing for exams

- **Azure OpenAI "On Your Data"** — directly wires AOAI models to a Cosmos MongoDB-vCore vector store (NoSQL vector integration also available).
- **Azure AI Foundry** — Cosmos DB appears as a first-class vector + agent-memory store when building agents.
- **Semantic Kernel / LangChain** — official connectors for both NoSQL and Mongo vCore vector stores.
- **Microsoft Fabric** — AI skills query Cosmos data via mirroring.

## Cosmos DB vs Azure AI Search (when asked)

| Dimension | Cosmos DB | AI Search |
|-----------|-----------|-----------|
| Primary role | Operational DB that also serves vectors | Dedicated search engine |
| Data freshness | Real-time (same container as writes) | Indexer pull (minutes) or push API |
| Hybrid search | ✅ vector + BM25 + RRF | ✅ vector + BM25 + semantic ranker + scoring profiles |
| Semantic ranker (LLM-powered) | ❌ (RRF only) | ✅ |
| Cost model | RU/s + storage | Units (replicas × partitions) |
| Best for | Unified app DB + RAG | Search-first apps, pre-existing indexed content |

Exam tells: if the scenario stresses **freshness**, **global distribution**, or **operational + vector in one**, pick Cosmos. If it stresses **semantic reranking**, **LLM-tuned ranking**, or **enterprise search over heterogeneous sources**, pick AI Search.

## Exam traps

- `EnableNoSQLVectorSearch` capability must be toggled at the account — not automatic.
- DiskANN is NoSQL **and** MongoDB vCore; HNSW is vCore-only.
- Vector properties must be **excluded from range indexing** to avoid 2× write RU cost.
- 2 000 dim / 16 000 dim — embedding models matter: `text-embedding-3-large` = 3072 dims; `text-embedding-3-small` = 1536 dims.
- Hybrid search needs **both** vector policy + full-text policy configured.

## Sources

- [Vector search in Cosmos DB NoSQL](https://learn.microsoft.com/en-us/azure/cosmos-db/vector-search)
- [Integrated vector database](https://learn.microsoft.com/en-us/azure/cosmos-db/vector-database)
- [Vector similarity overview](https://learn.microsoft.com/en-us/azure/cosmos-db/gen-ai/vector-search-overview)
- [MongoDB vCore vector search](https://learn.microsoft.com/en-us/azure/cosmos-db/mongodb/vcore/vector-search)
- [DiskANN + filtered vector GA (blog)](https://devblogs.microsoft.com/cosmosdb/diskann-in-azure-cosmos-db-for-mongodb/)
- [Vector, FTS, hybrid on NoSQL (blog)](https://devblogs.microsoft.com/cosmosdb/new-vector-search-full-text-search-and-hybrid-search-features-in-azure-cosmos-db-for-nosql/)
