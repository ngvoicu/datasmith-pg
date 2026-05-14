# CLAUDE.md — Data Smith

## Project Overview

Data Smith is a Claude Code plugin that provides expert PostgreSQL database architecture guidance. It has two layers:

1. **Plugin layer** — Markdown-based Claude Code plugin (skill + reference docs)
2. **Skill layer** — Universal SKILL.md that works with any AI coding tool via `npx skills add`

## Knowledge base

This repo currently has no neurons in the **ngvoicu-sme** brain. As you learn things worth keeping (plugin distribution decisions, schema-design patterns adopted by consuming projects, eval results), capture them via kluris (never edit brain files by hand — the skill enforces an approval protocol):

- `/kluris-ngvoicu-sme` — Claude Code skill (search, learn, remember, create)
- `kluris search "<query>" --brain ngvoicu-sme` — direct search

## Repository Structure

```
datasmith/
├── .claude-plugin/          # Plugin metadata
│   ├── plugin.json          # Name: datasmith-pg, version 0.1.0
│   └── marketplace.json     # Marketplace registration
├── references/              # Reference documentation
│   ├── patterns.md          # Common schema patterns (soft deletes, audit logs, multi-tenancy, trees)
│   ├── migration-guide.md   # Flyway conventions, zero-downtime migrations, seed data
│   ├── performance.md       # Index types, EXPLAIN ANALYZE, partitioning, concurrency
│   ├── anti-patterns.md     # Common database design mistakes and fixes
│   └── dialect-notes.md     # PostgreSQL vs MySQL vs SQLite vs Snowflake vs BigQuery
├── skills/
│   └── datasmith-pg/
│       └── SKILL.md         # → ../../SKILL.md (symlink for Claude Code plugin discovery)
├── SKILL.md                 # Universal skill definition (works with all tools)
└── README.md
```

## Architecture

### Plugin Layer

The plugin is consumed directly by Claude Code — no build step. Markdown files define behavior:

- **`plugin.json`** — Plugin identity (name: `datasmith-pg`, version: `0.1.0`)
- **`SKILL.md`** — Universal skill with PostgreSQL-first database architecture guidance. Defines natural language triggers (schema design, review, migration writing, normalization, indexing).

### Knowledge Base — `references/` Directory

Five reference documents provide deep-dive knowledge:

- **patterns.md** — Soft deletes, audit logs, multi-tenancy, RLS, state machines, trees, many-to-many, optimistic locking, JSONB, archive tables, partitioning, polymorphic associations, seed data, computed columns
- **migration-guide.md** — Flyway naming conventions, DDL strategy, zero-downtime patterns (expand/contract, batch updates, CONCURRENTLY indexes, safe NOT NULL, NOT VALID FKs)
- **performance.md** — B-Tree, GIN, BRIN, GiST, partial/covering indexes, EXPLAIN ANALYZE, partitioning strategies, query optimization, materialized views, advisory locks, VACUUM/ANALYZE
- **anti-patterns.md** — God tables, EAV, polymorphic associations, missing FKs, VARCHAR(255), FLOAT for money, bare TIMESTAMP, multi-valued strings, SERIAL, misused ENUMs, wide tables, missing FK indexes, N+1 queries, SELECT *
- **dialect-notes.md** — PostgreSQL, MySQL/MariaDB, SQLite, Snowflake, BigQuery, CockroachDB differences

## Key Conventions

### PostgreSQL-First Defaults

- `BIGINT GENERATED ALWAYS AS IDENTITY` over BIGSERIAL
- `TIMESTAMPTZ` over TIMESTAMP (always)
- Explicit constraint naming with prefixes (`pk_`, `fk_`, `uq_`, `chk_`, `idx_`, `trg_`)
- Store monetary values as integer cents or `NUMERIC(19,4)`
- Start at 3NF, denormalize only with evidence

### Cross-Tool Compatibility

The skill works with any AI coding tool that can read files:
- **Claude Code**: Full plugin support or skill via `npx skills add`
- **Codex**: Skill via `npx skills add`
- **Cursor / Windsurf / Cline**: Skill via `npx skills add`
- **Gemini CLI**: Skill via `npx skills add`

## Versions

- **Plugin**: v0.1.0 (`.claude-plugin/plugin.json`)
- **Author**: Gabriel Voicu

## Working on This Codebase

- SKILL.md is universal — it must work for all AI tools, not just Claude Code
- Reference files are deep-dive knowledge docs — keep them comprehensive but organized
- No build step — markdown files are consumed directly
- All supported tools use `npx skills add` for setup
- To test plugin changes locally: install the plugin in a test project (`claude plugin add /path/to/datasmith`), then verify schema design/review behavior
