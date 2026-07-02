# Azure Database for PostgreSQL — Study Notes

Quick-reference notes for exam prep and day-to-day work.

---

## 1. What Is It?

- **Fully managed** relational DB service based on **open-source PostgreSQL** (community edition → full compatibility with existing tools/apps).
- Microsoft manages: infrastructure, patching, backups, HA.
- You still control: maintenance windows, HA config, compute sizing.
- Good fit for AI apps: relational data + **JSONB** (flexible) + **pgvector** (vector search).

---

## 2. Architecture & Compute Tiers

- **Compute and storage are separated** → engine runs on a Linux VM, data on Azure managed storage.
  - Enables independent scaling + built-in durability (locally redundant storage replicas).

| Tier | VM Series | Best For |
|---|---|---|
| **Burstable** | B-series | Dev/test, small apps, intermittent workloads, cost-sensitive |
| **General Purpose** | D-series | Production, predictable/balanced workloads |
| **Memory Optimized** | E-series | High memory:vCPU ratio — caching, analytics, large in-memory sets |

> 💡 You can **change tiers after deployment** — requires a brief restart.

---

## 3. Managed Capabilities

### Backups
- **Automatic**, zone-redundant (if AZ region) or locally redundant storage.
- Default retention: **7 days**, extendable to **35 days**.
- Includes full snapshots + transaction logs → **point-in-time restore (PITR)** to any second in range.
- Encrypted with **AES 256-bit** (platform-managed keys by default, or customer-managed).

### Point-in-Time Restore
- Restores to a **new server instance** at a specified timestamp.
- Useful for accidental data loss or testing against historical state.

### PgBouncer (built-in connection pooler)
- Reduces overhead of new connections.
- Connect on **port 6432** (instead of standard **5432**).
- ⚠️ **Only available on General Purpose & Memory Optimized tiers — NOT Burstable.**
- Great for AI apps with many short-lived DB calls.

---

## 4. Connecting to PostgreSQL

### Endpoint format
```
<server-name>.postgres.database.azure.com
```

### Core connection parameters
- Host (FQDN)
- Port: `5432` (direct) or `6432` (PgBouncer)
- Database name
- Username (Entra auth: `username@servername`)
- Password (static or Entra token)
- SSL mode

### Authentication Methods

| Method | Notes |
|---|---|
| **Microsoft Entra ID** | OAuth 2.0 token-based; no password storage; supports managed identities; centralized identity + audit trail in Entra logs. Requires an Entra admin configured on the server. |
| **PostgreSQL native auth** | Username/password stored in DB; use for legacy apps, non-Entra identities, or local dev. Store secrets in **Key Vault**, rotate regularly. |

**Entra token flow:**
1. App requests token from Entra ID
2. Entra validates & returns token
3. App connects using token **as the password**
4. PostgreSQL validates token against configured Entra admin

```python
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()
token = credential.get_token("https://ossrdbms-aad.database.windows.net/.default")
# use token.token as the password
```
> `DefaultAzureCredential` tries managed identity → Azure CLI → other methods, in order.

### TLS/SSL
- **TLS required by default**; supports TLS 1.2 and 1.3 only (older versions rejected).

| `sslmode` | Behavior |
|---|---|
| `disable` | ❌ Rejected by Azure |
| `allow` | Encrypts if server requires; no cert validation |
| `prefer` | Encrypts if supported; no cert validation |
| `require` | Encryption enforced; no cert validation |
| `verify-ca` | Encryption + validates CA |
| `verify-full` | Encryption + CA + **hostname match** ✅ (recommended for production) |

### Connection String Examples
```
# Basic (SSL required)
postgresql://user:pass@myserver.postgres.database.azure.com/mydb?sslmode=require

# Full certificate verification
postgresql://user:pass@myserver.postgres.database.azure.com/mydb?sslmode=verify-full&sslrootcert=/etc/ssl/certs/ca-certificates.crt

# Through PgBouncer
postgresql://user:pass@myserver.postgres.database.azure.com:6432/mydb?sslmode=require
```

### Networking Models
- **Public access** — public IP + firewall rules (allow client IPs).
- **Private access** — private IP within VNet; reachable via VNet/peering/VPN/ExpressRoute. Common for production isolation.

### Key Extensions for AI Apps
| Extension | Purpose |
|---|---|
| `pgvector` | Vector data types + similarity search (embeddings) |
| `pg_trgm` | Trigram-based fuzzy text matching / autocomplete |
| `hstore` | Simple key-value data type |

Enable with: `CREATE EXTENSION <name>;`

### Decision Points
- **Compute tier**: Burstable (dev/cost) → General Purpose (prod, steady) → Memory Optimized (in-memory/analytics heavy).
- **Extensions**: Confirm availability early (vector search, full-text, geospatial).

---

## 5. Schema Design

### Hierarchy
```
Server → Databases → Schemas → Tables/Objects
```
- Default schema = `public`.
- **Separate databases**: full isolation, independent backup/restore.
- **Separate schemas**: logical grouping while still allowing FK joins across namespace.
- Most AI apps: single DB + default `public` schema is enough.

### Creating Tables
```sql
CREATE TABLE conversations (
    id BIGSERIAL PRIMARY KEY,
    session_id UUID NOT NULL,
    user_id VARCHAR(255) NOT NULL,
    started_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    ended_at TIMESTAMP WITH TIME ZONE,
    metadata JSONB DEFAULT '{}'::jsonb
);
```

**Primary Keys**
- `SERIAL` (32-bit) / `BIGSERIAL` (64-bit) → sequential integers, good for high insert volume.
- `UUID` with `DEFAULT gen_random_uuid()` → for external ID generation or merging multi-source data.
- Composite keys → when no single column is unique.

**Constraints**
- `NOT NULL`, `DEFAULT`, `CHECK (...)`, `UNIQUE`

### Key Data Types for AI Apps

| Type | Use When |
|---|---|
| `JSONB` | Variable structure, nested data, need indexing/queries |
| `TEXT` vs `VARCHAR(n)` | No perf difference in Postgres; use `VARCHAR(n)` to enforce max length |
| `TIMESTAMP WITH TIME ZONE` (`TIMESTAMPTZ`) | **Always** for temporal data — stored in UTC |
| `BYTEA` | Small binary blobs (use Blob Storage for large files) |
| `SERIAL` / `BIGSERIAL` | Auto-incrementing IDs (`BIGSERIAL` if >2B rows expected) |

### Relationships

**One-to-many** (e.g., conversation → messages):
```sql
CREATE TABLE messages (
    id BIGSERIAL PRIMARY KEY,
    conversation_id BIGINT NOT NULL REFERENCES conversations(id),
    role VARCHAR(50) NOT NULL CHECK (role IN ('user','assistant','system')),
    content TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

**Referential actions on delete/update:**
| Action | Effect |
|---|---|
| `RESTRICT` (default) | Blocks operation if references exist |
| `CASCADE` | Auto-deletes/updates referencing rows ⚠️ use carefully |
| `SET NULL` | Sets FK to NULL |
| `SET DEFAULT` | Sets FK to default value |

**Many-to-many** — use a junction table:
```sql
CREATE TABLE conversation_tags (
    conversation_id BIGINT REFERENCES conversations(id) ON DELETE CASCADE,
    tag_id BIGINT REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (conversation_id, tag_id)
);
```

### Indexes
- Default type: **B-tree** (equality + range queries).
- PostgreSQL auto-creates indexes for PKs and UNIQUE constraints.
- Create indexes on columns used in `WHERE`, `JOIN`, `ORDER BY`.
```sql
CREATE INDEX idx_messages_conversation_id ON messages(conversation_id);
CREATE INDEX idx_messages_conversation_created ON messages(conversation_id, created_at);
```
> ⚠️ Composite index column **order matters** — supports queries filtering on leading columns.
> Trade-off: indexes speed reads but slow writes + use disk space — be selective on write-heavy tables.

### Schema Management
- `ALTER TABLE ... ADD COLUMN / DROP COLUMN` — adding a column with a default is efficient (stored in system catalog, not rewritten per row).
- `DROP TABLE` — permanent; blocked by FK references unless `CASCADE`.
- DDL is **transactional** in PostgreSQL:
```sql
BEGIN;
ALTER TABLE conversations ADD COLUMN category VARCHAR(100);
CREATE INDEX idx_conversations_category ON conversations(category);
COMMIT;
```

---

## 6. Querying Data

### SQL Logical Execution Order (important for exams!)
| # | Clause | Purpose |
|---|---|---|
| 1 | `FROM` | Source tables |
| 2 | `WHERE` | Filter rows |
| 3 | `GROUP BY` | Group rows |
| 4 | `HAVING` | Filter groups |
| 5 | `SELECT` | Choose columns/expressions |
| 6 | `ORDER BY` | Sort |
| 7 | `LIMIT/OFFSET` | Restrict count |

**Alias scope rule:** Aliases from `SELECT` are only visible in clauses that run **after** it (`ORDER BY`, `LIMIT`) — NOT in `WHERE`/`GROUP BY`/`HAVING`.

```sql
-- FAILS: alias not yet defined at WHERE stage
SELECT DATE(created_at) AS msg_date, content
FROM messages
WHERE msg_date > '2024-01-01';   -- ❌ Error

-- WORKS: repeat expression
WHERE DATE(created_at) > '2024-01-01';

-- WORKS: ORDER BY runs after SELECT
... ORDER BY msg_date;
```

### PostgreSQL-Specific Filtering
- `ILIKE` → case-insensitive `LIKE` (`WHERE content ILIKE '%error%'`)
- `NULLS FIRST` / `NULLS LAST` → control NULL sort position
- `COALESCE(col, 'default')` → first non-null value

### JSONB Operators
| Operator | Returns | Example |
|---|---|---|
| `->` | JSON | `metadata->'status'` |
| `->>` | text | `metadata->>'status'` |
| `#>` | JSON (nested path) | `data#>'{results,0}'` |
| `#>>` | text (nested path) | `data#>>'{results,0,score}'` |
| `?` | key existence | `metadata ? 'priority'` |
| `@>` | containment | `data @> '{"status":"completed"}'` |

> `?` and `@>` can use **GIN indexes** for efficient filtering.

Expand JSON array:
```sql
SELECT DISTINCT c.*
FROM conversations c,
     jsonb_array_elements_text(c.metadata->'tags') AS tag
WHERE tag = 'support';
```

### Keyset (Cursor) Pagination — preferred over OFFSET at scale
- OFFSET pagination gets slower as page number grows (must scan+discard rows).
- Keyset pagination filters using last-seen values → **consistent performance**.
```sql
-- First page
SELECT id, content, created_at FROM messages
ORDER BY created_at DESC, id DESC LIMIT 20;

-- Next page (use last row's values)
SELECT id, content, created_at FROM messages
WHERE (created_at, id) < ('2024-06-15 10:30:00', 12345)
ORDER BY created_at DESC, id DESC LIMIT 20;
```
> Include `id` alongside timestamp to break ties.

### CTEs (Common Table Expressions)
```sql
WITH recent_conversations AS (
    SELECT id, user_id, started_at FROM conversations
    WHERE started_at > CURRENT_DATE - INTERVAL '7 days'
),
message_stats AS (
    SELECT conversation_id, COUNT(*) AS message_count
    FROM messages GROUP BY conversation_id
)
SELECT rc.user_id, COALESCE(ms.message_count, 0) AS message_count
FROM recent_conversations rc
LEFT JOIN message_stats ms ON rc.id = ms.conversation_id;
```

**Recursive CTE** — for hierarchical/tree data (task trees, threaded messages):
```sql
WITH RECURSIVE task_tree AS (
    SELECT id, parent_id, title, 1 AS depth
    FROM tasks WHERE id = 1
    UNION ALL
    SELECT t.id, t.parent_id, t.title, tt.depth + 1
    FROM tasks t
    INNER JOIN task_tree tt ON t.parent_id = tt.id
    WHERE tt.depth < 10          -- always cap depth to avoid infinite loop
)
SELECT * FROM task_tree ORDER BY depth, id;
```

### RETURNING clause
Retrieve generated values without a second query:
```sql
INSERT INTO conversations (user_id, session_id)
VALUES ('user123', 'sess_abc')
RETURNING id;

UPDATE tasks SET status='completed', completed_at=CURRENT_TIMESTAMP
WHERE id = 5
RETURNING id, status, completed_at;
```

### Upserts — `ON CONFLICT`
```sql
INSERT INTO user_preferences (user_id, preference_key, preference_value)
VALUES ('user123', 'theme', 'dark')
ON CONFLICT (user_id, preference_key)
DO UPDATE SET preference_value = EXCLUDED.preference_value,
              updated_at = CURRENT_TIMESTAMP;

-- Skip silently on conflict
INSERT INTO tags (name) VALUES ('important') ON CONFLICT (name) DO NOTHING;
```
- `EXCLUDED` = pseudo-table with the values that would've been inserted.
- Check insert-vs-update: `(xmax = 0) AS is_new` → `true` if newly inserted.

---

## 7. SDK Integration (Python — psycopg 3)

### Install
```bash
pip install "psycopg[binary]"
```
- `[binary]` = pre-compiled, no need for local libpq headers (dev convenience).
- Production compiling from source needs libpq dev headers.

### Connecting
```python
import psycopg

# Connection string
conn = psycopg.connect("postgresql://user:password@myserver.postgres.database.azure.com/mydb?sslmode=require")

# Individual parameters
conn = psycopg.connect(host="myserver.postgres.database.azure.com", dbname="mydb",
                        user="myuser", password="mypassword", sslmode="require")
```

### Context Managers (auto-cleanup)
```python
with psycopg.connect(connection_string) as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT * FROM conversations WHERE id = %s", (conversation_id,))
        row = cur.fetchone()
```
> ⚠️ Always use **parameterized queries** (`%s` / `%(name)s`) — never string concat — to prevent SQL injection.

### Fetching Results
- `fetchone()` → single row
- `fetchall()` → small result sets
- iterate cursor directly → large result sets (memory efficient)

### Timeouts & Retry
```python
conn = psycopg.connect(
    connection_string,
    connect_timeout=10,
    options="-c statement_timeout=30000"  # ms
)
```
- Web apps: 5–30s timeouts typical.
- Retry with exponential backoff on `OperationalError` (transient failures).
- **Don't retry** constraint violations or syntax errors — these need code fixes.
- Always close connections (context managers handle this).

### Error Handling
| Error | Cause | Action |
|---|---|---|
| Connection failure | Network/credentials/firewall/maintenance | Log details, show friendly message |
| `UniqueViolation` / `ForeignKeyViolation` / `CheckViolation` | Constraint breach | Roll back, give meaningful feedback |
| `DeadlockDetected` | Circular lock wait | Roll back & retry; design consistent lock ordering |
| `LockNotAvailable` | Lock wait timeout | Handle like deadlock |

### Performance Patterns
- **Batch inserts** instead of row-by-row loops.
- `executemany` → hundreds–few thousand rows.
- `COPY` → 10,000+ rows, **2–10x faster** than individual inserts:
```python
with cur.copy("COPY messages (conversation_id, role, content) FROM STDIN") as copy:
    for record in records:
        copy.write_row(record)
```
- **Prepared statements** auto-used by most drivers for parameterized queries → plan reused.
- **Connection pooling** — creating connections is expensive (handshake + auth + server resources):
```python
from psycopg_pool import ConnectionPool

pool = ConnectionPool(connection_string, min_size=1, max_size=10)

with pool.connection() as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT * FROM messages WHERE conversation_id = %s", (id,))
```

---

## 8. Quick Exam Cheat-Sheet

- ✅ **Public** vs **Private** access → firewall rules vs VNet.
- ✅ **PgBouncer port = 6432**; **NOT available on Burstable tier**.
- ✅ Default backup retention = **7 days**, max **35 days**.
- ✅ `verify-full` = the **most secure** sslmode (hostname + CA validation).
- ✅ Entra auth username format: `username@servername`.
- ✅ Alias defined in `SELECT` is usable in `ORDER BY`/`LIMIT` but **not** `WHERE`/`GROUP BY`/`HAVING`.
- ✅ `->>`/`#>>` return **text**; `->`/`#>` return **JSON**.
- ✅ Keyset pagination > OFFSET pagination for large tables.
- ✅ `ON CONFLICT ... DO UPDATE` uses `EXCLUDED` to reference incoming values.
- ✅ `COPY` is fastest for large bulk inserts (10k+ rows).
- ✅ Use `TIMESTAMPTZ`, not plain `TIMESTAMP`, for time-zone-safe data.
- ✅ Compute tiers: **Burstable (B)** → dev/test | **General Purpose (D)** → production | **Memory Optimized (E)** → in-memory/analytics.
