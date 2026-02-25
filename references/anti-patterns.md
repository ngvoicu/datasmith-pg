# SQL Anti-Patterns

Common database design mistakes and how to fix them. When reviewing schemas, scan for these
patterns and flag them immediately.

---

## The God Table

**Problem:** A single table with 50+ columns covering multiple concerns (user info, preferences,
billing, activity — all in one row). Leads to wide rows, wasted I/O, complex queries, and
locking contention.

**Fix:** Split into focused tables with proper relationships. One table per entity/concern,
joined by foreign keys.

```sql
-- Bad: user table with 80 columns
-- Good: user, user_preference, user_billing, user_activity
```

---

## Entity-Attribute-Value (EAV)

**Problem:** Storing data as generic key-value pairs (`entity_id, attribute, value`). Promises
"infinite flexibility" but delivers: impossible joins, no type safety, no constraints, no
foreign keys, and terrible query performance.

**Fix:** Use proper columns for known attributes. Use JSONB for truly dynamic metadata. If you
need a flexible schema, use JSONB with validation, not EAV.

```sql
-- Bad: EAV
SELECT value FROM attribute WHERE entity_id = 1 AND key = 'email';

-- Good: proper column
SELECT email FROM "user" WHERE id = 1;

-- Good: JSONB for genuinely variable data
SELECT metadata->>'custom_field' FROM "user" WHERE id = 1;
```

---

## Polymorphic Associations (parent_id + parent_type)

**Problem:** A `commentable_id` + `commentable_type` pattern that ORMs like Rails popularized.
It cannot be enforced with foreign keys, so referential integrity depends entirely on
application code. Typos in type strings, wrong ID spaces, and renamed entities silently
corrupt data.

**Fix:** Use exclusive belongs-to (separate FK columns with a CHECK constraint ensuring exactly
one is set) or separate junction tables per parent type. See `references/patterns.md` for the
full pattern.

---

## Missing Foreign Keys

**Problem:** Omitting FK constraints "for flexibility" or "performance." Without them, orphaned
rows accumulate silently, data integrity depends on perfect application code, and debugging
becomes much harder.

**Fix:** Always define foreign keys. The performance cost is negligible for most workloads (a
lookup on INSERT/UPDATE). If you truly need to skip FK checks for bulk loading, disable them
temporarily and re-enable after.

---

## VARCHAR(255) Everywhere

**Problem:** Using `VARCHAR(255)` as the default for every text column. The 255 limit is a MySQL
historical artifact (1-byte length prefix). It communicates no useful constraint and wastes
schema readability.

**Fix:** In PostgreSQL, use `TEXT` for unbounded strings (same performance as VARCHAR, no
arbitrary limit). Use `VARCHAR(n)` only when there's a real business constraint on length
(e.g., country code VARCHAR(2), phone VARCHAR(20)).

---

## FLOAT / DOUBLE for Monetary Values

**Problem:** Floating-point types introduce rounding errors. `0.1 + 0.2 != 0.3` in IEEE 754.
This causes silent miscalculations in financial totals, tax calculations, and reconciliation.

**Fix:** Use `NUMERIC(19,4)` for exact decimal arithmetic, or store values as integer cents
(`total_cents INTEGER`). Never use `FLOAT`, `DOUBLE PRECISION`, `REAL`, or PostgreSQL's `MONEY`
type for monetary values.

```sql
-- Bad
price DOUBLE PRECISION  -- 19.99 might be stored as 19.9899999...

-- Good (exact decimal)
price NUMERIC(19,4)     -- Exactly 19.9900

-- Good (integer cents — Stripe's approach)
price_cents INTEGER     -- 1999 = $19.99
```

---

## Bare TIMESTAMP (without Timezone)

**Problem:** `TIMESTAMP` (without TZ) stores a "wall clock time" with no timezone context. If
your server timezone changes, or data is queried from a different timezone, times silently shift.

**Fix:** Always use `TIMESTAMPTZ`. PostgreSQL stores it as UTC internally and converts to the
session timezone on display. This is always correct regardless of where the query runs.

```sql
-- Bad
created_at TIMESTAMP NOT NULL DEFAULT now()

-- Good
created_at TIMESTAMPTZ NOT NULL DEFAULT now()
```

---

## Multi-Valued String Columns

**Problem:** Storing multiple values in a single column separated by commas, pipes, or other
delimiters (`"red,blue,green"`). Makes querying, indexing, and updating individual values
extremely difficult.

**Fix:** Use a junction table for proper many-to-many relationships, or a PostgreSQL array
type (`TEXT[]`) if the values are simple and don't need their own attributes.

```sql
-- Bad
tags TEXT  -- "red,blue,green"

-- Good: junction table (when tags are entities)
CREATE TABLE product_tag (
  product_id  BIGINT NOT NULL REFERENCES product (id),
  tag_id      BIGINT NOT NULL REFERENCES tag (id),
  PRIMARY KEY (product_id, tag_id)
);

-- Acceptable: array type (when tags are simple strings)
tags TEXT[] NOT NULL DEFAULT '{}'
```

---

## SERIAL / BIGSERIAL (Legacy)

**Problem:** `SERIAL` and `BIGSERIAL` are PostgreSQL-specific shortcuts that create a separate
sequence object. They don't prevent manual inserts that desync the sequence, require extra
USAGE grants on the sequence, and are not SQL-standard.

**Fix:** Use `GENERATED ALWAYS AS IDENTITY` (PG 10+). It's SQL-standard, prevents accidental
manual value injection, and manages the sequence automatically with cleaner permissions.

```sql
-- Legacy
id BIGSERIAL PRIMARY KEY

-- Modern
id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
```

---

## PostgreSQL ENUM Types (When Misused)

**Problem:** PostgreSQL ENUM can become operationally painful when business values change often.
Adding and renaming values is supported, but removing/reordering values or larger refactors can
require careful type migration planning.

**Fix:** For fast-changing domains, prefer `TEXT` with a `CHECK` constraint (or a lookup table).
Use ENUM when the value set is stable and strict typing is valuable.

```sql
-- Better for frequently evolving states
status TEXT NOT NULL DEFAULT 'PENDING'
  CHECK (status IN ('PENDING', 'SHIPPED', 'DELIVERED'))

-- Better for stable states that benefit from strict type semantics
CREATE TYPE order_status AS ENUM ('PENDING', 'SHIPPED', 'DELIVERED');
status order_status NOT NULL DEFAULT 'PENDING'
```

---

## Wide Tables (100+ Columns)

**Problem:** Tables with excessive columns cause: large row sizes (I/O waste even when reading
few columns), frequent page splits, locking contention on updates, and cognitive overload for
developers.

**Fix:** Split into logical sub-tables joined by the same PK or a 1:1 relationship. Group
columns by access pattern — frequently-read columns together, rarely-accessed columns separate.

---

## Missing Indexes on Foreign Keys

**Problem:** PostgreSQL does NOT automatically create indexes on foreign key columns (unlike
MySQL). Without them, JOINs on FK columns do full table scans, and DELETE on the parent table
scans the child to check for references.

**Fix:** Always create an index on every FK column:

```sql
ALTER TABLE order_item ADD CONSTRAINT fk_order_item_order
  FOREIGN KEY (order_id) REFERENCES "order" (id);

-- Don't forget this!
CREATE INDEX idx_order_item_order ON order_item (order_id);
```

---

## N+1 Query Pattern

**Problem:** Fetching a list of parent records, then issuing one query per parent to fetch
children. 100 orders = 101 queries instead of 2.

**Fix:** Use JOINs, subqueries, or `IN` clauses to batch the child lookups:

```sql
-- Bad: 1 query for orders, then N queries for items
SELECT * FROM "order" WHERE customer_id = 42;
-- For each order: SELECT * FROM order_item WHERE order_id = ?;

-- Good: 2 queries total
SELECT * FROM "order" WHERE customer_id = 42;
SELECT * FROM order_item WHERE order_id IN (
  SELECT id FROM "order" WHERE customer_id = 42
);

-- Or: single query with JOIN
SELECT o.*, oi.*
FROM "order" o
JOIN order_item oi ON oi.order_id = o.id
WHERE o.customer_id = 42;
```

---

## SELECT * in Production Code

**Problem:** `SELECT *` fetches all columns, including large TEXT/JSONB fields you don't need.
It breaks when columns are added/removed, prevents index-only scans, and wastes network bandwidth.

**Fix:** Explicitly list the columns you need. Reserve `SELECT *` for ad-hoc queries and debugging.

---

## Quick Scan Checklist

When reviewing any schema, check for:

- [ ] All IDs use BIGINT (not INTEGER / SERIAL)
- [ ] All timestamps use TIMESTAMPTZ (not TIMESTAMP)
- [ ] All FKs have matching indexes
- [ ] All constraints are explicitly named
- [ ] All FKs have explicit ON DELETE behavior
- [ ] No FLOAT/DOUBLE for monetary values
- [ ] ENUM usage justified by value stability (CHECK preferred for evolving sets)
- [ ] No VARCHAR(255) without a business reason
- [ ] No polymorphic parent_id + parent_type
- [ ] No comma-separated multi-value columns
- [ ] created_at and updated_at on every table
- [ ] updated_at trigger defined
