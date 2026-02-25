# Performance Guide

## Index Type Selection

PostgreSQL offers several index types, each optimized for different query patterns:

### B-Tree (Default)

Best for: equality (`=`), range (`<`, `>`, `BETWEEN`), sorting (`ORDER BY`), and prefix matching (`LIKE 'foo%'`).

```sql
CREATE INDEX idx_order_created_at ON "order" (created_at DESC);
CREATE INDEX idx_user_email ON "user" (email);
```

B-Tree is the right choice 90% of the time. When in doubt, use B-Tree.

### GIN (Generalized Inverted Index)

Best for: multi-value columns — JSONB, arrays, full-text search (`tsvector`).

```sql
-- JSONB containment queries
CREATE INDEX idx_event_payload ON event USING GIN (payload);

-- Full-text search
CREATE INDEX idx_article_search ON article USING GIN (to_tsvector('english', title || ' ' || body));

-- Array containment
CREATE INDEX idx_product_tags ON product USING GIN (tags);
```

**Trade-off:** Slower writes (multiple index entries per row), faster reads. Avoid on
high-write tables unless the read benefit justifies it. For bulk inserts, drop the GIN index
and recreate it after.

### BRIN (Block Range Index)

Best for: very large tables with physically ordered data (time-series, append-only logs).

```sql
-- Extremely compact — stores min/max per block range, not per row
CREATE INDEX idx_event_log_created_at ON event_log USING BRIN (created_at);
```

**Trade-off:** Much smaller than B-Tree (orders of magnitude), but less precise — may scan
extra blocks. Only works well when data is physically correlated with the indexed column
(e.g., rows inserted in chronological order).

### GiST (Generalized Search Tree)

Best for: geometric data, range types, full-text search (alternative to GIN), nearest-neighbor.

```sql
-- Range type queries
CREATE INDEX idx_booking_period ON booking USING GIST (tstzrange(check_in, check_out));

-- Exclusion constraint (no overlapping bookings)
ALTER TABLE booking ADD CONSTRAINT excl_booking_overlap
  EXCLUDE USING GIST (room_id WITH =, tstzrange(check_in, check_out) WITH &&);
```

### Partial Indexes

Index only the rows you actually query. Dramatically smaller and faster:

```sql
-- Only index active orders (skip 90% of rows that are archived)
CREATE INDEX idx_order_active ON "order" (customer_id) WHERE status != 'ARCHIVED';

-- Only index non-deleted users
CREATE INDEX idx_user_active ON "user" (email) WHERE deleted_at IS NULL;

-- Only index current records (common in temporal data)
CREATE UNIQUE INDEX uq_subscription_current
  ON subscription (user_id) WHERE ended_at IS NULL;
```

### Covering Indexes (INCLUDE)

Add non-key columns to enable index-only scans (avoids heap fetch):

```sql
-- Queries that SELECT email, name WHERE id = X can be served entirely from the index
CREATE INDEX idx_user_lookup ON "user" (id) INCLUDE (email, name);
```

### Multi-Column Indexes

Column order matters. Lead with columns that match common WHERE prefixes and required ORDER BY
/ JOIN patterns; use selectivity as a tie-breaker. Confirm with `EXPLAIN ANALYZE`.

```sql
-- Query pattern: WHERE tenant_id = ? AND status = ? ORDER BY created_at DESC
CREATE INDEX idx_order_tenant_status_created_at
  ON "order" (tenant_id, status, created_at DESC);

-- Supports predicates that start with (tenant_id) or (tenant_id, status),
-- but not predicates that filter only on status
```

---

## Reading EXPLAIN ANALYZE

Run `EXPLAIN ANALYZE` to see the actual execution plan:

```sql
EXPLAIN ANALYZE
SELECT * FROM "order" WHERE customer_id = 42 AND status = 'PENDING';
```

### Key things to look for:

| Node Type | Meaning | Good or Bad? |
|-----------|---------|--------------|
| **Index Scan** | Uses an index to find rows | Good |
| **Index Only Scan** | Everything served from index (no heap fetch) | Best |
| **Bitmap Index Scan** | Uses index to build a bitmap, then fetches | Good for many rows |
| **Seq Scan** | Full table scan, reads every row | Bad on large tables (unless you need most rows) |
| **Nested Loop** | For each outer row, scans inner | Good for small result sets |
| **Hash Join** | Builds hash table, probes it | Good for medium-large joins |
| **Merge Join** | Both sides pre-sorted, merge | Good for large sorted datasets |

### Red flags:
- **Seq Scan on a large table** with a WHERE clause → missing index
- **Actual rows** much higher than **estimated rows** → run `ANALYZE` on the table
- **Sort** with high cost → consider an index that provides the ordering
- **Nested Loop** with many iterations → may need a Hash Join (check `work_mem`)

---

## Partitioning Strategies

Consider partitioning when a table exceeds ~100M rows or ~10GB.

### Range Partitioning (Most Common)

Best for time-series data:

```sql
CREATE TABLE metric (
  id          BIGINT GENERATED ALWAYS AS IDENTITY,
  sensor_id   BIGINT NOT NULL,
  value       NUMERIC(10,4) NOT NULL,
  recorded_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (recorded_at);

CREATE TABLE metric_2025_q1 PARTITION OF metric
  FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');
```

### List Partitioning

Best for categorical data with clear boundaries:

```sql
CREATE TABLE order_by_region (
  id      BIGINT GENERATED ALWAYS AS IDENTITY,
  region  TEXT NOT NULL,
  total   INT NOT NULL
) PARTITION BY LIST (region);

CREATE TABLE order_us   PARTITION OF order_by_region FOR VALUES IN ('US');
CREATE TABLE order_eu   PARTITION OF order_by_region FOR VALUES IN ('EU');
CREATE TABLE order_apac PARTITION OF order_by_region FOR VALUES IN ('APAC');
```

### Hash Partitioning

Best for even distribution when no natural range/list exists:

```sql
CREATE TABLE session_data (
  id      UUID PRIMARY KEY,
  data    JSONB
) PARTITION BY HASH (id);

CREATE TABLE session_data_0 PARTITION OF session_data FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE session_data_1 PARTITION OF session_data FOR VALUES WITH (MODULUS 4, REMAINDER 1);
-- etc.
```

### Partitioning Tips
- Keep partitions uniformly sized for consistent maintenance
- Too many partitions (1000+) hurts query planning
- Use `pg_partman` extension to automate partition creation/maintenance
- Indexes must be created on each partition (or use the parent table and PG auto-creates them)

---

## Query Optimization Patterns

### Avoid SELECT *
Select only the columns you need. This reduces I/O and enables index-only scans.

### Use CTEs Wisely
In PostgreSQL 12+, CTEs are inlined (optimized) by default unless they have side effects or
are referenced multiple times. Use `MATERIALIZED` to force evaluation if needed:

```sql
-- Force CTE to evaluate once (useful if referenced many times)
WITH expensive_calc AS MATERIALIZED (
  SELECT user_id, complex_function(data) AS result
  FROM big_table
)
SELECT ...
```

### Materialized Views for Expensive Reports

```sql
CREATE MATERIALIZED VIEW mv_monthly_revenue AS
SELECT
  date_trunc('month', created_at) AS month,
  SUM(total_cents) AS revenue_cents,
  COUNT(*) AS order_count
FROM "order"
WHERE status = 'DELIVERED'
GROUP BY 1;

-- Refresh periodically (takes a lock, but fast)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_monthly_revenue;
```

`CONCURRENTLY` requires a UNIQUE index on the materialized view.

### Batch Operations

For large INSERT/UPDATE/DELETE operations, work in batches to avoid long transactions,
excessive WAL generation, and lock contention:

```sql
-- Delete old records in batches
DELETE FROM event_log
WHERE id IN (
  SELECT id FROM event_log WHERE created_at < '2023-01-01' LIMIT 10000
);
-- Repeat until 0 rows deleted
```

---

## Concurrency and Locking

### Transaction Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Use Case |
|-------|-----------|-------------------|-------------|----------|
| Read Committed (default) | No | Possible | Possible | Most applications |
| Repeatable Read | No | No | No* | Reports needing consistent snapshot |
| Serializable | No | No | No | Financial transactions, critical integrity |

*PostgreSQL's Repeatable Read actually prevents phantom reads too (it uses snapshot isolation).

### Advisory Locks

For application-level locking without table locks:

```sql
-- Session-level lock (held until session ends or explicitly released)
SELECT pg_advisory_lock(hashtext('process_daily_report'));
-- ... do work ...
SELECT pg_advisory_unlock(hashtext('process_daily_report'));

-- Transaction-level lock (released at COMMIT/ROLLBACK)
SELECT pg_advisory_xact_lock(hashtext('unique_operation_key'));
```

### Deadlock Prevention
1. **Consistent lock ordering** — Always acquire locks on tables/rows in the same order
2. **Keep transactions short** — Don't hold locks while waiting for user input or external APIs
3. **Use `lock_timeout`** — Set `SET lock_timeout = '5s'` to fail fast instead of waiting
4. **Use `SKIP LOCKED`** — For queue-like patterns where contention is expected:

```sql
-- Worker picks next available job without blocking
SELECT * FROM job_queue
WHERE status = 'PENDING'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

---

## Maintenance

### VACUUM and ANALYZE

PostgreSQL's MVCC model creates dead rows on UPDATE/DELETE. VACUUM reclaims this space.

- **autovacuum** handles this automatically in most cases — ensure it's enabled
- Run `ANALYZE` after bulk data loads to update query planner statistics
- Monitor bloat with `pgstattuple` or `pg_stat_user_tables.n_dead_tup`

### Connection Pooling

PostgreSQL creates a process per connection (~10MB each). For applications with many
connections, use a pooler:

- **PgBouncer** — Lightweight, battle-tested. Transaction-mode pooling recommended.
- **pgcat** — Modern alternative with sharding support.
- **Built-in** — PG 14+ has some basic connection reuse, but external poolers are still preferred.

Rule of thumb: keep active connections under `max_connections` (default 100). Scale with
pooling, not by raising `max_connections`.
