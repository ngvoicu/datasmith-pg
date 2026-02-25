# Data Smith

**Your AI-powered PostgreSQL database architect.**

Data Smith turns any AI coding tool into a senior database engineer. It designs production-grade schemas from scratch, reviews and optimizes existing schemas, writes complex SQL (JOINs, CTEs, window functions), generates Flyway migrations, and advises on normalization, indexing, and multi-tenancy.

Works with Claude Code (as a plugin), Codex, Cursor, Windsurf, Cline, Gemini CLI, and any AI coding tool that can read files.

## The Problem

AI coding tools are great at writing application code but produce mediocre database schemas. Common issues:

- **Missing constraints** — No explicit naming, no CHECK constraints, no ON DELETE behavior
- **Timezone bugs** — Using `TIMESTAMP` instead of `TIMESTAMPTZ`
- **Legacy patterns** — `BIGSERIAL` instead of `GENERATED ALWAYS AS IDENTITY`
- **No indexing strategy** — Foreign keys without indexes, no partial indexes
- **`VARCHAR(255)` everywhere** — Cargo-culted from MySQL defaults
- **FLOAT for money** — Rounding errors waiting to happen
- **Over-engineering** — EAV tables, polymorphic associations, premature denormalization

Data Smith fixes all of this. It enforces PostgreSQL best practices while keeping schemas simple and maintainable.

## What It Covers

### Schema Design from Scratch

Give it a business domain and it produces:
- Clean, normalized tables (singular snake_case, explicit naming)
- Named constraints (`fk_order_user`, `chk_amount_positive`, `uq_user_email`)
- Proper foreign keys with explicit `ON DELETE` behavior
- Indexes on every FK column, query-pattern indexes, partial indexes
- Auto-updating `updated_at` triggers
- Multi-tenancy with composite foreign keys when needed

### Schema Review

Paste existing SQL and it scans for:
- `TIMESTAMP` without TZ (bug risk)
- `SERIAL`/`BIGSERIAL` (legacy, should be IDENTITY)
- Missing `created_at`/`updated_at`
- Foreign keys without indexes
- Nullable columns that should be NOT NULL
- `VARCHAR(255)` everywhere
- FLOAT/DOUBLE for monetary values
- Auto-generated constraint names
- Normalization violations (1NF through BCNF)

### Complex SQL

Writes production-quality queries with:
- CTEs for readable multi-step logic
- Window functions (ROW_NUMBER, SUM OVER, LAG/LEAD)
- Proper column aliasing and table qualification
- `NULLS LAST` in ORDER BY for nullable columns

### Migration Guidance

Flyway-compatible migrations with:
- `V{version}__{Description}.sql` naming
- Zero-downtime patterns (expand/contract, `CONCURRENTLY` indexes)
- Safe NOT NULL additions (default first, then constraint)
- `NOT VALID` foreign keys validated separately
- Batch updates for large tables

### Reference Knowledge Base

Five deep-dive reference documents:

| Document | What It Covers |
|----------|---------------|
| `patterns.md` | Soft deletes, audit logs, multi-tenancy, RLS, state machines, trees, many-to-many, optimistic locking, JSONB, partitioning, computed columns |
| `migration-guide.md` | Flyway conventions, zero-downtime strategies, seed data, dangerous operations checklist |
| `performance.md` | B-Tree/GIN/BRIN/GiST indexes, EXPLAIN ANALYZE, partitioning, materialized views, advisory locks, VACUUM |
| `anti-patterns.md` | God tables, EAV, polymorphic associations, missing FKs, VARCHAR(255), FLOAT for money, N+1 queries |
| `dialect-notes.md` | PostgreSQL vs MySQL vs SQLite vs Snowflake vs BigQuery vs CockroachDB |

## Installation

Two ways to use Data Smith, depending on your setup.

### Path 1: Claude Code Plugin (Recommended)

Full plugin with SKILL.md auto-triggers and all reference documentation.

```bash
# In Claude Code, run:
/plugin marketplace add ngvoicu/datasmith
/plugin install datasmith-pg
```

Or manually:
```bash
git clone https://github.com/ngvoicu/datasmith.git ~/.claude/plugins/datasmith
```

After install, just describe your schema needs:
```
"design a schema for an e-commerce app with orders, products, and customers"
"review this SQL for anti-patterns"
"write a Flyway migration to add a soft delete column"
```

### Path 2: Quick Setup via npx (Any Tool)

Installs the SKILL.md into your tool's skill/instruction directory.

```bash
# Claude Code (skill only — auto-triggers)
npx skills add ngvoicu/datasmith -a claude-code

# OpenAI Codex
npx skills add ngvoicu/datasmith -a codex

# Cursor
npx skills add ngvoicu/datasmith -a cursor

# Windsurf
npx skills add ngvoicu/datasmith -a windsurf

# Cline
npx skills add ngvoicu/datasmith -a cline

# Gemini CLI
npx skills add ngvoicu/datasmith -a gemini
```

### Comparison: Plugin vs npx

| Feature | Plugin (full) | npx (any tool) |
|---------|:---:|:---:|
| Auto-triggers on schema/SQL tasks | Yes | Yes |
| Reference docs (patterns, migrations, performance, anti-patterns, dialects) | Yes | No |
| Works with Codex, Cursor, Windsurf, etc. | No | Yes |
| Multi-tool compatibility | Yes | Yes |

## Usage Examples

### Design a Schema

```
"model a SaaS project management app with workspaces, projects, tasks, and team members"
```

Data Smith produces:
1. Entity-relationship analysis
2. Full DDL with named constraints, indexes, triggers
3. Multi-tenancy setup with composite foreign keys
4. Open questions for your confirmation

### Review Existing SQL

```sql
-- Paste your schema
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255),
  email VARCHAR(255),
  created_at TIMESTAMP DEFAULT now()
);
```

Data Smith flags:
- `SERIAL` should be `BIGINT GENERATED ALWAYS AS IDENTITY`
- `VARCHAR(255)` should be `TEXT` (PostgreSQL has no performance difference)
- `TIMESTAMP` should be `TIMESTAMPTZ`
- Missing `NOT NULL` on `name` and `email`
- Missing `UNIQUE` on `email`
- Missing `updated_at` column and trigger
- Auto-generated primary key name should be `pk_user`

### Write Complex Queries

```
"find the top 5 customers by total spend in the last 30 days, with their order count and average order value"
```

### Generate Migrations

```
"write a Flyway migration to add soft deletes to the order table with zero downtime"
```

## Core Principles

1. **Simple > Clever** — A schema a junior dev can understand beats a clever one no one maintains
2. **Explicit > Implicit** — `user_id` not `uid`, named constraints not auto-generated
3. **Constraints are documentation** — NOT NULL, UNIQUE, FK, CHECK communicate intent
4. **Name every constraint** — `fk_order_user` not `order_user_id_fkey`
5. **Start normalized, denormalize with evidence** — 3NF is the right default
6. **Design for evolution** — Safe, additive, reversible migrations

## Standard Columns

Every entity table starts with:

```sql
id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
```

**Why IDENTITY over BIGSERIAL?** SQL standard (PG 10+), prevents accidental manual inserts, fewer grants needed.

**Why BIGINT?** INTEGER overflows at ~2.1B rows. BIGINT avoids painful future migrations.

**Why TIMESTAMPTZ?** Bare `TIMESTAMP` causes bugs when servers change timezone or queries span regions.

## Constraint Naming

| Type | Prefix | Pattern | Example |
|------|--------|---------|---------|
| Primary Key | `pk_` | `pk_{table}` | `pk_order` |
| Foreign Key | `fk_` | `fk_{table}_{referenced}` | `fk_order_item_order` |
| Unique | `uq_` | `uq_{table}_{column(s)}` | `uq_user_email` |
| Check | `chk_` | `chk_{table}_{description}` | `chk_order_amount_positive` |
| Index | `idx_` | `idx_{table}_{column(s)}` | `idx_order_created_at` |
| Trigger | `trg_` | `trg_{table}_{action}` | `trg_order_updated_at` |

## Multi-Tool Support

The skill is pure markdown. Claude Code, Codex, Cursor, Windsurf, Cline, and Gemini CLI all benefit from the same PostgreSQL expertise.

```bash
npx skills add ngvoicu/datasmith -a <tool>
```

## Project Structure

```
datasmith/
├── .claude-plugin/
│   ├── plugin.json                 # Plugin metadata (v0.1.0)
│   └── marketplace.json            # Marketplace registration
├── references/
│   ├── patterns.md                 # Common schema patterns
│   ├── migration-guide.md          # Flyway + zero-downtime migrations
│   ├── performance.md              # Indexes, EXPLAIN, partitioning
│   ├── anti-patterns.md            # Common mistakes and fixes
│   └── dialect-notes.md            # Cross-database differences
├── skills/
│   └── datasmith-pg/
│       └── SKILL.md                # → ../../SKILL.md (symlink)
├── SKILL.md                        # Universal skill (works with all tools)
└── README.md
```

## License

MIT
