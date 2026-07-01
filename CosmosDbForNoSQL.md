# Azure Cosmos DB for NoSQL — Exam & Reference Notes

---

## 1. Resource Hierarchy

```
Account → Database → Container → Item
```

| Level | Purpose | Key Detail |
|---|---|---|
| **Account** | Top-level unit | DNS: `https://<name>.documents.azure.com:443/` |
| **Database** | Logical namespace | Max 500 DBs + containers per account |
| **Container** | Scalability unit | Requires partition key; schema-agnostic |
| **Item** | JSON document | Requires `id` property |

---

## 2. Partition Keys ⚠️ Most Critical Decision

### Good Partition Key
- Exists in **every** document
- **High cardinality** (many distinct values)
- Distributes data evenly
- Aligns with query patterns

### Examples
- User data → `userId`
- Product catalog → `categoryId`
- Multitenant app → `tenantId`

### Bad Partition Keys (avoid)
- **Low cardinality** → e.g., `isActive` (only 2 values = 2 partitions)
- **Time-based** → e.g., date fields create hot partitions during peak hours

> ⚠️ **You cannot change partition key after container creation** — must migrate data to a new container.

---

## 3. Throughput (RU/s)

### Request Units (RUs)
- Unified metric for CPU + memory + IOPS
- Every operation returns RU cost in `x-ms-request-charge` header

### Typical RU Costs
| Operation | ~Cost |
|---|---|
| Point read (1 KB item) | ~1 RU |
| Write (1 KB item) | ~5–10 RU |
| Simple filtered query | ~3–5 RU |
| Complex aggregate query | Hundreds of RU |

### Throughput Levels
- **Database-level** → shared across containers (good for variable traffic)
- **Container-level** → dedicated (predictable performance)

### Throughput Modes
| Mode | Behavior | Minimum |
|---|---|---|
| **Manual** | Fixed RU/s you set | 400 RU/s |
| **Autoscale** | Scales 10%–100% of max | 1,000 RU/s max (scales down to 100) |

### RU Cost Factors
- Item size (larger = more RUs)
- Indexing policy (more indexes = higher write cost)
- Consistency level (stronger = more RUs)
- Query complexity
- Cross-partition queries cost more than single-partition

---

## 4. System Properties (auto-added by Cosmos DB)

| Property | Description |
|---|---|
| `_rid` | Internal resource identifier |
| `_self` | Unique URI for the item |
| `_etag` | Entity tag for **optimistic concurrency** |
| `_ts` | Unix timestamp of last modification |
| `_attachments` | Legacy attachment path |

> 💡 Use `_etag` to prevent lost updates — if item changed since you read it, update fails with conflict error.

---

## 5. SDK — Authentication

### Option A: Account Key (simple, less secure)
```python
client = CosmosClient(endpoint, credential=key)
```

### Option B: Microsoft Entra ID / RBAC (recommended for production)
```python
from azure.identity import DefaultAzureCredential
client = CosmosClient(endpoint, credential=DefaultAzureCredential())
```

`DefaultAzureCredential` auto-selects:
- Local dev → Azure CLI / VS credentials
- Deployed → Managed Identity

### Built-in RBAC Roles
- `Cosmos DB Built-in Data Reader` → read only
- `Cosmos DB Built-in Data Contributor` → read + write

---

## 6. SDK — Key Patterns

### ✅ Always Reuse CosmosClient (singleton)
```python
# Create ONCE at app startup
cosmos_service = CosmosService("https://myaccount.documents.azure.com:443/")
```
Creating a new client per request = wasted connections + added latency.

### Navigate Resources
```python
database = client.get_database_client("mydb")
container = database.get_container_client("mycontainer")
# Note: no network call yet — just a handle
```

### Idempotent Creation (use these in startup scripts)
```python
database = client.create_database_if_not_exists(id="mydb")
container = database.create_container_if_not_exists(
    id="products",
    partition_key=PartitionKey(path="/categoryId")
)
```

---

## 7. SDK — CRUD Operations

| Method | Behavior | Fails if... |
|---|---|---|
| `create_item()` | Insert new | Item already exists (409) |
| `upsert_item()` | Insert or replace | — (safe for caching) |
| `replace_item()` | Update existing | Item doesn't exist |
| `read_item()` | Point read by id + partition key | Item not found |
| `delete_item()` | Remove item | Item not found |

### Optimistic Concurrency with `_etag`
```python
item = container.read_item(item="id-123", partition_key="electronics")
item["price"] = 149.99
container.replace_item(item=item["id"], body=item, if_match=item["_etag"])
# Fails with CosmosAccessConditionFailedError if item changed since read
```

---

## 8. Query Language (SQL-like)

### Basic Syntax
```sql
SELECT p.id, p.name, p.price
FROM products p
WHERE p.categoryId = "electronics" AND p.price < 500
```

### Useful Operators & Functions
| Feature | Example |
|---|---|
| Range | `p.price BETWEEN 50 AND 200` |
| String contains | `CONTAINS(p.name, "Speaker")` |
| Multiple values | `p.categoryId IN ("electronics", "appliances")` |
| Array check | `ARRAY_CONTAINS(p.features, "wifi")` |
| Sort | `ORDER BY p.price DESC` |

### Aggregates
```sql
SELECT COUNT(1) as total, AVG(p.price) as avg, MIN(p.price) as min, MAX(p.price) as max
FROM products p WHERE p.categoryId = "electronics"
```
Supported: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`

### VALUE keyword — returns scalars/arrays without wrapper object
```sql
SELECT VALUE p.name FROM products p WHERE p.categoryId = "electronics"
-- Returns: ["Smart Speaker", "Wireless Headphones"]
```

---

## 9. Query Optimization

### Single-Partition Query (cheaper, faster)
```python
items = container.query_items(
    query=query,
    parameters=parameters,
    partition_key="electronics"  # routes to ONE partition
)
```

### Cross-Partition Query (when needed)
```python
items = container.query_items(
    query=query,
    enable_cross_partition_query=True  # fans out to all partitions
)
```

### Always Use Parameterized Queries
```python
query = "SELECT * FROM p WHERE p.categoryId = @cat AND p.price < @max"
parameters = [
    {"name": "@cat", "value": "electronics"},
    {"name": "@max", "value": 500.00}
]
```
Prevents injection + enables query plan caching.

### Pagination
```python
query_iterable = container.query_items(query=query, max_item_count=50)
for page in query_iterable.by_page():
    for item in page:
        process(item)
```

---

## 10. Monitor RU Consumption

```python
headers = container.client_connection.last_response_headers
ru_cost = headers['x-ms-request-charge']
activity_id = headers['x-ms-activity-id']  # use for support tickets
```

---

## 11. Quick-Reference: Cost Optimization Checklist

- [ ] Use **point reads** (`read_item`) instead of queries when you know `id` + partition key
- [ ] **Project only needed fields** in SELECT (not `SELECT *`)
- [ ] **Include partition key** in WHERE clause or `partition_key` parameter
- [ ] Use **parameterized queries** for repeated queries
- [ ] Apply **filters early** to reduce scanned items
- [ ] Use `TOP` to limit results when you don't need everything
- [ ] Use **autoscale** for variable/unpredictable traffic
- [ ] Use **shared throughput** at database level for containers with variable load
- [ ] Monitor `x-ms-request-charge` to spot expensive operations

---

## 12. Common Exceptions to Handle

| Exception | Cause |
|---|---|
| `CosmosResourceExistsError` | `create_item()` on duplicate |
| `CosmosResourceNotFoundError` | Read/delete on missing item |
| `CosmosAccessConditionFailedError` | `_etag` mismatch (concurrent update) |

---

*Source: Microsoft Learn — Azure Cosmos DB for NoSQL*


# Azure Cosmos DB — Vector Search Study Notes

Quick-reference notes for exam prep and real-world work with vector embeddings in **Azure Cosmos DB for NoSQL**.

---

## 1. Core Concepts

- **Embeddings** = numerical (float) representations of text/images/data, generated by models like Azure OpenAI's `text-embedding-ada-002` (1,536 dims) or `text-embedding-3-large` (3,072 dims).
- Similar meaning → vectors close together in vector space (even with different wording, e.g. "WiFi" vs "wireless connection").
- Cosmos DB lets you store the embedding **as a property inside the same document** as your business data — no separate vector DB needed.
- **Must enable vector search on the account first**:
  - Azure Portal → Settings → Features → *Vector Search for NoSQL API*
  - CLI: `az cosmosdb update --capabilities EnableNoSQLVectorSearch`
  - Auto-approved, may take ~15 minutes.

---

## 2. Document Design

Store embedding as an array property alongside metadata:

```json
{
  "id": "doc-12345",
  "title": "Troubleshooting wireless network connections",
  "content": "...",
  "category": "networking",
  "productId": "router-x100",
  "createdDate": "2024-06-15T10:30:00Z",
  "embedding": [0.023, -0.041, 0.067, ...]
}
```

---

## 3. Vector Policy (⚠️ set at container creation — CANNOT be changed later)

| Field | Meaning | Default |
|---|---|---|
| `path` | JSON path to embedding property (required) | — |
| `dataType` | `float32`, `float16`, `int8`, `uint8` | `float32` |
| `dimensions` | Must match embedding model output | 1536 |
| `distanceFunction` | `cosine`, `dotproduct`, `euclidean` | `cosine` |

```python
vector_embedding_policy = {
    "vectorEmbeddings": [
        {"path": "/embedding", "dataType": "float32",
         "distanceFunction": "cosine", "dimensions": 1536}
    ]
}
```

**Multiple embeddings per document** (e.g. title + content) → just add more entries to `vectorEmbeddings`.

### Distance Functions — when to use which
- **cosine** → default choice for text embeddings; ignores magnitude; scores -1 to +1 (practically 0–1). ✅ Recommended for Azure OpenAI embeddings.
- **dotproduct** → mathematically same as cosine *if* vectors are normalized (Azure OpenAI vectors are); slightly faster.
- **euclidean** → straight-line distance; 0 = identical, higher = more different; niche use cases where magnitude matters.

### Data Types — storage vs precision trade-off
- `float32` → full precision, most storage. **Best default for most apps.**
- `float16` → ~50% storage savings, minimal quality loss.
- `int8` / `uint8` → quantized, big savings, needs careful accuracy testing.

---

## 4. Indexing Policy (vector indexes)

Exclude the embedding path from normal indexing, add a dedicated vector index:

```python
indexing_policy = {
    "indexingMode": "consistent",
    "automatic": True,
    "includedPaths": [{"path": "/*"}],
    "excludedPaths": [
        {"path": "/\"_etag\"/?"},
        {"path": "/embedding/*"}     # embeddings don't benefit from range index
    ],
    "vectorIndexes": [
        {"path": "/embedding", "type": "diskANN"}
    ]
}
```

### Vector Index Types

| Type | Search | Max Dims | Best For |
|---|---|---|---|
| `flat` | Exact brute-force, 100% recall | 505 | Small datasets, exact results |
| `quantizedFlat` | Compressed brute-force | 4,096 | Up to ~50,000 vectors/partition |
| `diskANN` | Approximate (DiskANN algorithm) | 4,096 | Large datasets (millions of vectors), best performance |

⚠️ **`quantizedFlat` and `diskANN` need ≥1,000 vectors** to be effective — below that, queries fall back to full scan (higher RU cost).

---

## 5. Creating a Container (both policies set together, at creation time)

```python
container = database.create_container(
    id="knowledge-base",
    partition_key=PartitionKey(path="/category"),
    indexing_policy=indexing_policy,
    vector_embedding_policy=vector_embedding_policy
)
```
- Pick partition key based on common query filter patterns (e.g. `/category`).

## 6. Inserting Documents

```python
response = openai_client.embeddings.create(input=document_text, model="text-embedding-ada-002")
embedding = response.data[0].embedding
document["embedding"] = embedding
container.upsert_item(document)
```
- Use `upsert_item` — handles insert + update.
- **Regenerate embedding whenever content changes** to keep it in sync.

---

## 7. Querying with `VectorDistance`

```sql
SELECT TOP 10
    c.id, c.title, c.category,
    VectorDistance(c.embedding, @queryVector) AS SimilarityScore
FROM c
ORDER BY VectorDistance(c.embedding, @queryVector)
```

**Key rules:**
- Always convert the *user's search text* into an embedding using **the same model** used for documents (different models = incompatible vector spaces).
- Always use **`TOP N`** — without it, query scans everything (high RU cost/latency).
- Use **parameterized queries** (`@queryVector`) — enables query plan caching, avoids unwieldy query strings.

### `VectorDistance` parameters
1. `vector_expr_1` — document's embedding path
2. `vector_expr_2` — query vector
3. `bool_expr` (optional) — `true` forces brute-force search (default `false`)
4. `obj_expr` (optional) — override options like `distanceFunction`/`dataType`

### Similarity Score Interpretation (cosine)
| Score | Meaning |
|---|---|
| +1.0 | Identical |
| 0.7–0.9 | Highly similar |
| 0.5–0.7 | Moderately similar |
| 0.1–0.5 | Low similarity |
| ≤0.0 | Dissimilar |

Common threshold: **0.7** for high relevance, **0.5** for broader recall.

### Choosing TOP N by scenario
- **RAG apps** → 5–10 results
- **User-facing search** → 10–20 (with pagination)
- **Recommendations** → 3–5

### Indexed vs. Brute-force
- Default = uses vector index (fast, approximate, high recall).
- Force exact search: `VectorDistance(c.embedding, @queryVector, true)`
- Brute-force = compares against every document → higher RU, higher latency, poor scaling. Use only for testing/small datasets/when 100% accuracy required.

---

## 8. Combining Filters with Vector Search

```sql
SELECT TOP 10 c.id, c.title, VectorDistance(c.embedding, @queryVector) AS SimilarityScore
FROM c
WHERE c.category = @category AND c.createdDate > @startDate
ORDER BY VectorDistance(c.embedding, @queryVector)
```

- **Pre-filtering**: filters applied before vector comparison → happens when filter is selective (removes most docs) → improves performance.
- **Post-filtering**: filters applied after ranking → may return **fewer results than TOP N** if top matches get filtered out.
- Optimizer picks automatically based on filter selectivity + indexes.

**Common filter patterns:**
```sql
WHERE c.documentType = 'troubleshooting-guide'
WHERE c.createdDate >= '2024-01-01' AND c.createdDate < '2025-01-01'
WHERE c.productId IN ('router-x100', 'router-x200')
WHERE ARRAY_CONTAINS(c.accessGroups, @userGroup)
WHERE c.status = 'published'
```

- Include partition key in WHERE clause + pass `partition_key=` param → routes query to single partition → less RU, less latency.

---

## 9. Hybrid Search (Vector + Full-Text) — RRF

Use when exact terms (error codes, product names) AND semantic meaning both matter.

**Requires:**
1. Full-text policy:
```json
{ "defaultLanguage": "en-US", "fullTextPaths": [{"path": "/content", "language": "en-US"}] }
```
2. Indexing policy with both:
```json
{
  "vectorIndexes": [{"path": "/embedding", "type": "diskANN"}],
  "fullTextIndexes": [{"path": "/content"}]
}
```

**Query using `ORDER BY RANK RRF(...)`:**
```sql
SELECT TOP 10 *
FROM c
ORDER BY RANK RRF(
    VectorDistance(c.embedding, @queryVector),
    FullTextScore(c.content, @searchTerm1, @searchTerm2)
)
```

**Weighting** (array order matches function order):
```sql
ORDER BY RANK RRF(VectorDistance(...), FullTextScore(...), [2, 1])
```
- Higher vector weight → prioritize semantic/natural language.
- Higher full-text weight → prioritize exact keyword/code matches.
- Equal weights → good general-purpose default.

**Multi-vector search** (e.g., title + content embeddings) also uses RRF:
```sql
ORDER BY RANK RRF(
    VectorDistance(c.titleEmbedding, @queryVector),
    VectorDistance(c.contentEmbedding, @queryVector)
)
```

---

## 10. Performance Optimization Checklist

- ✅ Choose right index: `diskANN` for >50K vectors/partition, `quantizedFlat` for smaller sets.
- ✅ Always limit with smallest reasonable `TOP N`.
- ✅ Filter/query by partition key when possible.
- ✅ Project only needed fields (avoid `SELECT *`).
- ✅ Monitor RU cost via `x-ms-request-charge` header or Azure Portal.
- ✅ Hybrid search costs more (2 scoring functions) — use only when it adds value.
- ✅ Index all properties used in WHERE clauses.
- ✅ Test with production-scale data — performance differs a lot at scale.

---

## 11. Keeping Embeddings Fresh — Change Feed

**Change feed** = persistent, ordered log of inserts/updates per partition. Enabled by default, no config needed.

Used to auto-regenerate embeddings when source content changes.

### Push vs Pull Model

| | Push Model | Pull Model |
|---|---|---|
| How | Auto-delivered (e.g. Azure Functions trigger) | App polls for changes |
| Pros | No polling, auto partition mgmt & checkpointing, less code | More control, lower resource use, good for batch/one-time jobs |
| Best for | Continuous/production embedding refresh | Migrations, infrequent batch refresh |

### Azure Functions Trigger Example
```python
@app.cosmos_db_trigger(
    arg_name="documents",
    container_name="knowledge-base",
    database_name="support-db",
    connection="CosmosDBConnection",
    lease_container_name="leases",
    create_lease_container_if_not_exists=True
)
def refresh_embeddings(documents: func.DocumentList):
    for doc in documents:
        if should_refresh_embedding(doc):
            # regenerate embedding & upsert
            ...
```

### Lease Container
- Separate container required by the change feed processor.
- Tracks: **checkpoints** (resume after failure), **partition ownership** (scaling), **failover** (other instances take over on failure).
- Small throughput (~400 RU/s) usually enough.
- Must have a **different name** than the data container.

### Selective Refresh (avoid unnecessary API cost)
- Compare changed fields (`title`, `content`, etc.) — skip metadata-only changes (status, category).
- Better: store a **content hash** (e.g. SHA-256) and compare hash instead of full field values.

### High-Volume Handling
- **Batch API calls**: send array of texts in one embeddings request.
- **Rate limiting/backoff**: retry with exponential backoff on API limits.
- **Queue-based processing**: decouple detection from generation via Azure Queue Storage.
- **Prioritized refresh**: process high-priority docs first.

### Pull Model (Python SDK)
```python
container.query_items_change_feed(start_time="Beginning")
```
- Save the **continuation token** to resume later without reprocessing/missing changes.

### Idempotency & Error Handling
- Same change may be delivered more than once → processing logic must be idempotent (regenerating same embedding from same content = safe).
- Handle: deleted docs (`CosmosResourceNotFoundError`), concurrent updates (use ETags if needed), API failures (retry / dead-letter queue).

### Scaling
- Azure Functions scales automatically based on backlog — monitor `ChangeFeedProcessorHostLag`.
- Manual deployments: add/remove instances; lease container coordinates partition distribution.

---

## 🎯 Exam-Style Quick Facts

- Vector policy **cannot** be modified after container creation.
- Vector search only works on **new** containers.
- `TOP N` is **required** in vector queries for performance.
- Same embedding model **must** be used for both documents and queries.
- `flat` index max dims = **505**; `quantizedFlat`/`diskANN` max = **4,096**.
- `diskANN`/`quantizedFlat` need **≥1,000 vectors** to be effective.
- Cosine similarity range: **-1 to +1** (practical: 0 to 1).
- Hybrid search combines vector + full-text using **RRF** (`ORDER BY RANK RRF(...)`).
- Change feed is **enabled by default**, requires a separate **lease container** for processing.
