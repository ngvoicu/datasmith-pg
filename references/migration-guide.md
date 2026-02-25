# Migration Guide

## Flyway Naming Conventions

Flyway migration files follow a strict naming pattern:

```
V{version}__{Description}.sql     -- Versioned (applied once, in order)
U{version}__{Description}.sql     -- Undo (rollback for the matching version)
R__{Description}.sql              -- Repeatable (re-applied when checksum changes)
```

**Version formats** (pick one and stick with it):

| Strategy | Example | Best For |
|----------|---------|----------|
| Sequential | `V1__Create_user_table.sql` | Small teams, simple projects |
| Date-based | `V2025.02.22.1__Create_user_table.sql` | Large teams (reduces merge conflicts) |
| Semantic | `V001.001__Create_user_table.sql` | Projects with maintenance branches |

**Rules:**
- Double underscore `__` separates version from description (required)
- Underscores in description become spaces in Flyway's log
- Versions must be unique and monotonically increasing
- Never modify a migration after it's been applied to any environment

---

## Migration Structure

Organize migrations by concern:

```
src/main/resources/db/migration/
├── common/                          -- Shared across all environments
│   ├── V1__Initial_schema.sql       -- Tables, constraints, functions, triggers
│   ├── V2__Add_payment_table.sql
│   └── V3__Add_status_index.sql
├── dev/                             -- Dev/staging seed data only
│   ├── V100__Seed_test_users.sql
│   └── V101__Seed_demo_data.sql
└── R__Create_views.sql              -- Repeatable: views, functions that may change
```

---

## Writing Safe Migrations

### DDL Strategy: Fail-Fast vs Idempotent

For Flyway **versioned** migrations, prefer deterministic fail-fast DDL so unexpected state is
detected immediately. Use idempotent guards only when re-runs are intentional (bootstrap setup,
repeatable objects, environment provisioning).

| Scenario | Preferred style | Why |
|----------|-----------------|-----|
| Versioned migration (`V...`) | Fail-fast (`CREATE TABLE ...`) | Detects schema drift early |
| Bootstrap/repeatable script (`R...`) | Idempotent (`IF [NOT] EXISTS`) | Safe to re-run intentionally |

```sql
-- Versioned migration (fail fast)
CREATE TABLE customer_note (
  id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_id  BIGINT NOT NULL,
  note         TEXT NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  CONSTRAINT fk_customer_note_customer
    FOREIGN KEY (customer_id) REFERENCES customer (id) ON DELETE CASCADE
);

CREATE INDEX idx_customer_note_customer_id ON customer_note (customer_id);
```

```sql
-- Bootstrap/repeatable script (idempotent)
CREATE TABLE IF NOT EXISTS user_preference (
  id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id     BIGINT NOT NULL,
  key         TEXT NOT NULL,
  value       TEXT,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),

  CONSTRAINT fk_user_preference_user FOREIGN KEY (user_id) REFERENCES "user" (id) ON DELETE CASCADE,
  CONSTRAINT uq_user_preference_key UNIQUE (user_id, key)
);

CREATE INDEX IF NOT EXISTS idx_user_preference_user ON user_preference (user_id);

-- Columns (no IF NOT EXISTS for ALTER TABLE in older PG versions — check first)
DO $$
BEGIN
  IF NOT EXISTS (
    SELECT 1 FROM information_schema.columns
    WHERE table_name = 'user' AND column_name = 'phone'
  ) THEN
    ALTER TABLE "user" ADD COLUMN phone TEXT;
  END IF;
END $$;
```

### Idempotent Seed Data

Use `ON CONFLICT` for seed data that may be re-run:

```sql
INSERT INTO role (id, name, description)
OVERRIDING SYSTEM VALUE
VALUES
  (1, 'ADMIN', 'Full access'),
  (2, 'MANAGER', 'Team management'),
  (3, 'EMPLOYEE', 'Standard access')
ON CONFLICT (id) DO NOTHING;

-- Reset sequence so future inserts don't conflict
SELECT setval(pg_get_serial_sequence('role', 'id'), 3, true);
```

---

## Zero-Downtime Migration Strategies

### The Golden Rules

1. **Never rename a column/table** that the application currently uses
2. **Never drop a column** until the application no longer reads it
3. **Never add a NOT NULL column** without a DEFAULT (it locks the table for a full rewrite on older PG)
4. **Never hold an exclusive lock** during an expensive operation

### Expand and Contract Pattern

The safest approach for breaking changes:

**Phase 1 — Expand** (deploy migration, application reads both old and new):
```sql
-- Add new column alongside old one
ALTER TABLE "user" ADD COLUMN full_name TEXT;

-- Backfill in batches (not one giant UPDATE)
UPDATE "user" SET full_name = first_name || ' ' || last_name
WHERE id BETWEEN 1 AND 10000;
-- Repeat for next batch...
```

**Phase 2 — Migrate** (deploy application code that writes to new column):
- Application writes to both `full_name` AND `first_name`/`last_name`
- Application reads from `full_name`

**Phase 3 — Contract** (deploy migration to remove old columns):
```sql
ALTER TABLE "user" DROP COLUMN first_name;
ALTER TABLE "user" DROP COLUMN last_name;
ALTER TABLE "user" ALTER COLUMN full_name SET NOT NULL;
```

### Batch Updates for Large Tables

Never update millions of rows in a single transaction. Execute smaller batches repeatedly and
commit between batches from the caller (migration runner, job worker, or script).

```sql
-- Run this statement repeatedly until it updates 0 rows
WITH batch AS (
  SELECT id
  FROM "order"
  WHERE new_status IS NULL
  ORDER BY id
  LIMIT 10000
)
UPDATE "order" o
SET new_status = o.old_status
FROM batch
WHERE o.id = batch.id;
```

`COMMIT` must be issued by the caller between executions. Do not put transaction control inside
`DO $$ ... $$` blocks.

### Safe Index Creation

Creating indexes on large tables locks the table. Use `CONCURRENTLY`:

```sql
-- Does NOT lock the table (but takes longer and can't run inside a transaction)
CREATE INDEX CONCURRENTLY idx_order_customer ON "order" (customer_id);
```

**Important:** `CREATE INDEX CONCURRENTLY` cannot run inside a transaction block. In Flyway,
use a separate migration file for each concurrent index.

---

## Adding Constraints Safely

### Adding a NOT NULL constraint

On PostgreSQL 11+, adding a column with a DEFAULT is instant (no table rewrite):

```sql
-- Instant on PG 11+ (stores default in catalog, doesn't rewrite table)
ALTER TABLE "order" ADD COLUMN priority TEXT NOT NULL DEFAULT 'NORMAL';
```

For older versions or adding NOT NULL to an existing column:
```sql
-- Step 1: Add as nullable
ALTER TABLE "order" ADD COLUMN priority TEXT DEFAULT 'NORMAL';
-- Step 2: Backfill in batches
-- Step 3: Set NOT NULL after all rows have values
ALTER TABLE "order" ALTER COLUMN priority SET NOT NULL;
```

### Adding a Foreign Key without locking

```sql
-- Add FK as NOT VALID (instant, doesn't check existing rows)
ALTER TABLE "order"
  ADD CONSTRAINT fk_order_customer
  FOREIGN KEY (customer_id) REFERENCES customer (id)
  NOT VALID;

-- Validate existing rows separately (takes a SHARE UPDATE EXCLUSIVE lock, not ACCESS EXCLUSIVE)
ALTER TABLE "order" VALIDATE CONSTRAINT fk_order_customer;
```

---

## Dangerous Operations Checklist

Operations that can cause downtime on large tables:

| Operation | Risk | Safe Alternative |
|-----------|------|-----------------|
| `ALTER TABLE ADD COLUMN ... NOT NULL` (no DEFAULT, PG < 11) | Full table rewrite | Add nullable, backfill, then set NOT NULL |
| `CREATE INDEX` (without CONCURRENTLY) | Locks table for writes | `CREATE INDEX CONCURRENTLY` |
| `ALTER TABLE ... SET DATA TYPE` | Full table rewrite | Add new column, backfill, drop old |
| Single UPDATE on millions of rows | Long lock, WAL bloat | Batch updates with commits between statements |
| `ALTER TABLE ... ADD CONSTRAINT ... FOREIGN KEY` | Scans entire table | Add with `NOT VALID`, then `VALIDATE` separately |
| `DROP TABLE` / `DROP COLUMN` in use | Application errors | Remove from app first, then drop |

---

## Common Pitfalls

1. **Modifying applied migrations** — Flyway validates checksums. Changing an applied migration
   breaks deployments. Create a new migration instead.
2. **Long-running transactions in migrations** — They hold locks and prevent autovacuum. Keep
   transactions short and batch large operations.
3. **Blind `IF NOT EXISTS` in versioned migrations** — It can hide schema drift and failed
   assumptions. Use it only when re-runs are intentional.
4. **Forgetting sequence resets after seed data** — If you insert rows with hardcoded IDs,
   reset the sequence or future inserts will conflict.
5. **Running DDL and DML in the same migration** — Separate them. DDL changes need to be
   committed before DML can reference them in some edge cases.

---

## Tools

| Tool | Purpose |
|------|---------|
| **Flyway** | Migration versioning and execution |
| **pgroll** | Zero-downtime, reversible schema migrations (serves multiple schema versions) |
| **pg-osc** | Online schema changes for large tables |
| **reshape** | Zero-downtime migrations with automatic expand/contract |
| **Alembic** | Python-native migration tool (SQLAlchemy ecosystem) |
| **Prisma Migrate** | TypeScript/JavaScript schema-first migrations |
