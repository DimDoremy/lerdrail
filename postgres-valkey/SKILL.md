---
name: postgres-valkey
description: Use when designing database schema or cache/queue logic for a Laravel app on PostgreSQL + Redis/Valkey. Enforces the project's first data-architecture principle (a wide-table + JsonB middle form designed to migrate bidirectionally toward normalized SQL or ClickHouse, via tiered modeling and unified UUID v7 primary keys). Covers PG feature decisions (JSONB, arrays, enums, GIN/expression indexes, full-text search, CTEs) and Redis/Valkey usage (cache, queue, Horizon, locks, rate limiting) with the Lerd host-name convention. Defers live schema inspection and query execution to Laravel Boost MCP.
---

# PostgreSQL & Redis/Valkey Data Layer

## The First Principle (read first)

Every table in this project follows the **wide-table + JsonB middle form**, designed for **bidirectional migration**: toward normalized SQL (promote JsonB keys to columns) or toward ClickHouse (keep append-only wide tables, ORDER BY time+UUID). Two hard constraints make it work:

1. **Tiered modeling** — core business entities are normalized (FKs, JOINs, soft-deletes); event/log/behavioral streams are append-only wide tables (no UPDATE/DELETE, time-indexed, ClickHouse-first).
2. **Unified UUID v7 primary keys** on all new tables — time-ordered (good for PG B-tree *and* ClickHouse ORDER BY), globally unique (no merge clashes during phased migration), one type everywhere.

This is non-negotiable for new tables. Full rationale, the tier classification test, JsonB rules, and both migration paths in [references/data-architecture.md](references/data-architecture.md). Decide every table against the [decision checklist](references/data-architecture.md#decision-checklist-for-a-new-table) before writing a migration.

## How this skill relates to other layers

- **Live schema inspection / running queries** → defer to **Boost MCP** (`schema`, `database queries` tools). Don't re-derive schema from files.
- **Eloquent model/cast/migration code patterns** → `laravel-conventions` (especially [models-and-casts.md](../laravel-conventions/references/models-and-casts.md), which implements the UUID v7 base class and JsonB casts defined here).
- **Running migrations / DB snapshots / psql** → Lerd (`lerd artisan migrate`, `lerd db:shell`, `lerd db:snapshot`).
- **PG / Valkey / ClickHouse API specifics** → context7 (`/websites/postgresql_current`, `/websites/valkey_io`, `/clickhouse/clickhouse-docs`).

## PostgreSQL features — use them, that's why we're here

PostgreSQL was chosen for its JSONB/array/enum/indexing strengths, which make the middle form natural. When designing, lean on these instead of workarounds:

- **JsonB** for restless payloads — GIN index (open-ended queries) or expression index (one hot key). [postgres-features.md](references/postgres-features.md)
- **Arrays** (`text[]`, `int[]`) for simple multi-value columns (tags, role lists) — cheaper than a join table when you don't need FK integrity.
- **Native enums** (`CREATE TYPE`) for fixed vocabularies — `$table->enum()` in migration, or a real PG enum type for performance/clarity.
- **Full-text search** (`tsvector`, `tsquery`, GIN) before reaching for Meilisearch — often enough.
- **Indexes**: GIN (JsonB/FTS/array), B-tree (default), **partial** (`WHERE`), **expression** (on `(col->>'k')`).
- **CTEs / window functions** for analytics queries (`WITH ...`, `OVER (PARTITION BY ...)`).

Details, exact DDL, and Laravel `DB::statement`/raw-query patterns in [references/postgres-features.md](references/postgres-features.md).

## Migration templates

Two canonical shapes, both UUID v7:

- **Tier 1 — core entity** (normalized): `$table->uuid('id')->primary()` + FKs + `timestamps()` + `softDeletes()`. [migration-templates.md](references/migration-templates.md)
- **Tier 2 — event stream** (append-only): UUID v7 + indexed `created_at` + JsonB `payload` + GIN, **no** softDeletes, no UPDATE/DELETE.

Both templates, with copy-paste migration code and matching factories, in [references/migration-templates.md](references/migration-templates.md).

## The ClickHouse exit

Tier 2 streams are the first ClickHouse candidates. Type mapping (`uuid→UUID`, `jsonb→Object('json')`, `timestamptz→DateTime64`), `ORDER BY (created_at, id)` design, and append-only conversion in [references/clickhouse-migration.md](references/clickhouse-migration.md). Query ClickHouse docs via context7 `/clickhouse/clickhouse-docs`.

## Redis / Valkey

Wire-compatible; Laravel config is identical either way. Inside the Lerd FPM container the host is **`lerd-redis`** (not `127.0.0.1` — see [lerd-dev host-networking](../lerd-dev/references/host-networking.md)). `lerd env` writes it; `lerd service start redis` starts it.

Use it for:
- **Cache** — `Cache::store('redis')`, tags, lock-aware atomic operations.
- **Queue** — `QUEUE_CONNECTION=redis`; run workers via `lerd queue:start` / Horizon `lerd horizon:start`.
- **Locks / rate limiting** — `Cache::lock()`, `RateLimiter`.

Full config, Valkey-as-Redis-replacement notes, and Horizon under Lerd in [references/cache-queue.md](references/cache-queue.md).

## Quick reference

| Want to… | Do |
|----------|----|
| Classify a new table | [decision checklist](references/data-architecture.md#decision-checklist-for-a-new-table) |
| Write a Tier 1 migration | [migration-templates.md](references/migration-templates.md#tier-1--core-entity) |
| Write a Tier 2 (event stream) migration | [migration-templates.md](references/migration-templates.md#tier-2--event-stream-append-only) |
| Index a JsonB hot key | expression index `(meta->>'category')` — [postgres-features.md](references/postgres-features.md) |
| Add full-text search | `tsvector` + GIN — [postgres-features.md](references/postgres-features.md) |
| Inspect live schema | Boost `schema` tool |
| Run a migration | `lerd artisan migrate` |
| Snapshot before risky change | `lerd db:snapshot <name>` |
| Start Redis / queue / Horizon | `lerd service start redis` / `lerd queue:start` / `lerd horizon:start` |

## Common mistakes

| Mistake | Fix |
|---------|-----|
| `$table->id()` (bigint) on a new table | Use `$table->uuid('id')->primary()` + `HasUuids` trait |
| Event-stream table with `softDeletes()` | Remove it — append-only; supersede with a new row |
| JsonB key you `WHERE` on every query | Promote to a real column + expression index |
| `REDIS_HOST=127.0.0.1` | `lerd env` → `REDIS_HOST=lerd-redis` |
| Reaching for Meilisearch for 1k rows | PG full-text search (`tsvector` + GIN) is probably enough |
| Modeling everything normalized | Streams belong in append-only wide tables (Tier 2) |
| Modeling everything as JsonB blobs | Hot query fields and FKs must be real columns |
