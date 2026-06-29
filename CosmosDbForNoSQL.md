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
