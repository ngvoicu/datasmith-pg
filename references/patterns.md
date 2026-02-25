# Common Schema Patterns

## Soft Deletes

Instead of physically deleting rows, mark them as deleted. Use when you need audit trails,
undo functionality, or referential integrity with historical records.

```sql
ALTER TABLE "user" ADD COLUMN deleted_at TIMESTAMPTZ;

-- Query for active users
SELECT * FROM "user" WHERE deleted_at IS NULL;

-- Partial index for performance (only indexes non-deleted rows)
CREATE INDEX idx_user_active ON "user" (id) WHERE deleted_at IS NULL;
```

**Trade-off:** Soft deletes complicate every query (you must always filter `WHERE deleted_at IS NULL`).
For large datasets where deletes are rare, consider the archive table pattern instead.

---

## Audit Log Table

Track every change to sensitive tables:

```sql
CREATE TABLE audit_log (
  id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  table_name  TEXT NOT NULL,
  record_id   BIGINT NOT NULL,
  operation   TEXT NOT NULL CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE')),
  old_data    JSONB,
  new_data    JSONB,
  changed_by  BIGINT,
  changed_at  TIMESTAMPTZ NOT NULL DEFAULT now(),

  CONSTRAINT fk_audit_log_changed_by FOREIGN KEY (changed_by) REFERENCES "user" (id)
);

CREATE INDEX idx_audit_log_table_record ON audit_log (table_name, record_id);
CREATE INDEX idx_audit_log_changed_at ON audit_log (changed_at DESC);
```

**Automation:** Use a trigger function to populate the audit log automatically on INSERT/UPDATE/DELETE.

---

## Multi-Tenancy (Row-Level Isolation)

Add `tenant_id` (or `company_id`, `organization_id`) to every tenant-scoped table. Enforce
isolation with composite foreign keys so data can never cross tenant boundaries:

```sql
CREATE TABLE project (
  id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  tenant_id   BIGINT NOT NULL,
  name        TEXT NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),

  CONSTRAINT fk_project_tenant FOREIGN KEY (tenant_id) REFERENCES tenant (id) ON DELETE RESTRICT
);

CREATE INDEX idx_project_tenant ON project (tenant_id);

-- Composite unique so child tables can reference (id, tenant_id)
CREATE UNIQUE INDEX uq_project_id_tenant ON project (id, tenant_id);

-- Child table enforces same-tenant relationship via composite FK
CREATE TABLE task (
  id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  project_id  BIGINT NOT NULL,
  tenant_id   BIGINT NOT NULL,
  title       TEXT NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),

  CONSTRAINT fk_task_project_tenant
    FOREIGN KEY (project_id, tenant_id) REFERENCES project (id, tenant_id) ON DELETE CASCADE,
  CONSTRAINT fk_task_tenant
    FOREIGN KEY (tenant_id) REFERENCES tenant (id) ON DELETE RESTRICT
);

CREATE INDEX idx_task_project ON task (project_id);
CREATE INDEX idx_task_tenant ON task (tenant_id);
```

### Row-Level Security (PostgreSQL)

For additional defense-in-depth, enable RLS so even direct SQL access is tenant-scoped:

```sql
ALTER TABLE project ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON project
  USING (tenant_id = current_setting('app.current_tenant_id')::BIGINT);
```

The application sets `app.current_tenant_id` at the start of each request/connection.

---

## Status / State Machine

For status fields that evolve over time, a TEXT column with a CHECK constraint is the most
flexible default — values can be added or removed with a simple ALTER. For stable, well-known
value sets where strict typing is valuable, a PostgreSQL ENUM is a valid alternative:

```sql
CREATE TABLE "order" (
  id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  status      TEXT NOT NULL DEFAULT 'PENDING'
              CHECK (status IN ('PENDING', 'CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED', 'REFUNDED')),
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Index for filtering by status
CREATE INDEX idx_order_status ON "order" (status);

-- Partial index for the most common filter
CREATE INDEX idx_order_pending ON "order" (id) WHERE status = 'PENDING';
```

**CHECK vs ENUM trade-off:** CHECK constraints are flexible and portable — values can be
added or removed with a simple constraint swap. PostgreSQL ENUMs are compact (4 bytes) and
type-safe, but removing or renaming values requires careful type migration. Choose based on
how often you expect the value set to change.

---

## Tree / Hierarchical Data (Adjacency List)

Simple and effective for shallow trees (categories, org charts, up to ~5 levels):

```sql
CREATE TABLE category (
  id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  parent_id   BIGINT,
  name        TEXT NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),

  CONSTRAINT fk_category_parent FOREIGN KEY (parent_id) REFERENCES category (id) ON DELETE SET NULL
);

CREATE INDEX idx_category_parent ON category (parent_id);

-- Recursive CTE to get full hierarchy
WITH RECURSIVE category_tree AS (
  SELECT id, name, parent_id, 0 AS depth
  FROM category WHERE parent_id IS NULL

  UNION ALL

  SELECT c.id, c.name, c.parent_id, ct.depth + 1
  FROM category c
  JOIN category_tree ct ON ct.id = c.parent_id
)
SELECT * FROM category_tree ORDER BY depth, name;
```

For deep trees (> 5 levels) or frequent subtree queries, consider the **ltree** extension
or a closure table pattern.

---

## Many-to-Many (Junction Table)

```sql
CREATE TABLE user_role (
  user_id     BIGINT NOT NULL,
  role_id     BIGINT NOT NULL,
  granted_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  granted_by  BIGINT,

  CONSTRAINT pk_user_role PRIMARY KEY (user_id, role_id),
  CONSTRAINT fk_user_role_user FOREIGN KEY (user_id) REFERENCES "user" (id) ON DELETE CASCADE,
  CONSTRAINT fk_user_role_role FOREIGN KEY (role_id) REFERENCES role (id) ON DELETE CASCADE,
  CONSTRAINT fk_user_role_granted_by FOREIGN KEY (granted_by) REFERENCES "user" (id) ON DELETE SET NULL
);

-- The composite PK covers lookups by user_id. Add explicit index for role_id lookups.
CREATE INDEX idx_user_role_role ON user_role (role_id);
```

---

## Optimistic Locking

Prevents lost updates in concurrent systems without holding database locks:

```sql
ALTER TABLE "order" ADD COLUMN version INT NOT NULL DEFAULT 1;

-- Application logic:
UPDATE "order"
SET status = 'CONFIRMED', version = version + 1, updated_at = now()
WHERE id = $1 AND version = $2;

-- If 0 rows updated → someone else modified it → retry or error
```

The application reads the current `version`, then uses it in the WHERE clause of the UPDATE.
If another transaction modified the row in between, the UPDATE affects 0 rows and the
application knows to retry.

---

## JSONB for Flexible Metadata

Use JSONB for truly variable data — not as an excuse to avoid proper columns:

```sql
CREATE TABLE event (
  id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  type        TEXT NOT NULL,
  payload     JSONB NOT NULL DEFAULT '{}',
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- GIN index for querying inside JSONB
CREATE INDEX idx_event_payload ON event USING GIN (payload);

-- Query patterns
SELECT * FROM event WHERE payload->>'user_id' = '123';
SELECT * FROM event WHERE payload @> '{"status": "failed"}';
```

**Rule of thumb:** If you query a JSONB field in WHERE clauses frequently, it probably belongs
in its own column instead.

---

## Archive Table Pattern

For large datasets where soft deletes cause bloat, move deleted rows to a separate table:

```sql
CREATE TABLE order_archive (LIKE "order" INCLUDING ALL);

-- Move old completed orders to archive
WITH moved AS (
  DELETE FROM "order"
  WHERE status = 'DELIVERED' AND updated_at < now() - INTERVAL '2 years'
  RETURNING *
)
INSERT INTO order_archive SELECT * FROM moved;
```

---

## Partitioning (Range — Time-Series)

For tables that grow very large (>100M rows or >10GB), partition by time range:

```sql
CREATE TABLE event_log (
  id          BIGINT GENERATED ALWAYS AS IDENTITY,
  event_type  TEXT NOT NULL,
  payload     JSONB,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE event_log_2025_01 PARTITION OF event_log
  FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
CREATE TABLE event_log_2025_02 PARTITION OF event_log
  FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

-- Automate partition creation with pg_partman or a cron job
```

Queries that filter on `created_at` automatically prune irrelevant partitions. See
`references/performance.md` for detailed partitioning strategies.

---

## Polymorphic Association (Done Right)

**Avoid** the `parent_id` + `parent_type` anti-pattern — it cannot be enforced with foreign keys.

Instead, use **exclusive belongs-to** with separate nullable FK columns:

```sql
CREATE TABLE comment (
  id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  -- Exclusive belongs-to: exactly one must be non-null
  post_id     BIGINT,
  photo_id    BIGINT,
  video_id    BIGINT,
  body        TEXT NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),

  CONSTRAINT fk_comment_post  FOREIGN KEY (post_id)  REFERENCES post (id)  ON DELETE CASCADE,
  CONSTRAINT fk_comment_photo FOREIGN KEY (photo_id) REFERENCES photo (id) ON DELETE CASCADE,
  CONSTRAINT fk_comment_video FOREIGN KEY (video_id) REFERENCES video (id) ON DELETE CASCADE,

  -- Ensure exactly one parent is set
  CONSTRAINT chk_comment_single_parent CHECK (
    (CASE WHEN post_id  IS NOT NULL THEN 1 ELSE 0 END +
     CASE WHEN photo_id IS NOT NULL THEN 1 ELSE 0 END +
     CASE WHEN video_id IS NOT NULL THEN 1 ELSE 0 END) = 1
  )
);
```

If the number of parent types is large or dynamic, use **separate junction tables** instead
(one per parent type), each with proper foreign keys.

---

## Idempotent Seed Data

For development/staging seed data that can be re-run safely:

```sql
-- Insert with conflict handling
INSERT INTO role (id, name, description)
OVERRIDING SYSTEM VALUE
VALUES
  (1, 'ADMIN', 'Full system access'),
  (2, 'MANAGER', 'Team management'),
  (3, 'EMPLOYEE', 'Standard access')
ON CONFLICT (id) DO NOTHING;

-- Reset sequence to avoid conflicts with future inserts
SELECT setval(pg_get_serial_sequence('role', 'id'), 3, true);
```

Note: `OVERRIDING SYSTEM VALUE` is required when inserting explicit values into IDENTITY columns.

---

## Dynamic Timestamp Triggers

Set timestamps conditionally based on state transitions:

```sql
CREATE OR REPLACE FUNCTION set_completed_at()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.status = 'COMPLETED' AND (OLD.status IS DISTINCT FROM 'COMPLETED') THEN
    NEW.completed_at = now();
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_task_completed_at
  BEFORE UPDATE ON task
  FOR EACH ROW EXECUTE FUNCTION set_completed_at();
```

---

## Generated / Computed Columns

For values derived from other columns, use PostgreSQL generated columns (PG 12+):

```sql
CREATE TABLE product (
  id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  brand           TEXT NOT NULL,
  model           TEXT NOT NULL,
  price_cents     INT NOT NULL,
  quantity        INT NOT NULL,
  total_cents     INT GENERATED ALWAYS AS (price_cents * quantity) STORED,
  full_name       TEXT GENERATED ALWAYS AS (brand || ' ' || model) STORED
);
```

Generated columns are automatically maintained by PostgreSQL and can be indexed.
