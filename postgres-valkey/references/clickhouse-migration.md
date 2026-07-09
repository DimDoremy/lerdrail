# PostgreSQL → ClickHouse Migration

## Table of Contents
- [When](#when)
- [What migrates first](#what-migrates-first)
- [Type mapping](#type-mapping)
- [ORDER BY and engine choice](#order-by-and-engine-choice)
- [Append-only conversion](#append-only-conversion)
- [Ingestion](#ingestion)
- [Querying from Laravel](#querying-from-laravel)
- [Do not migrate](#do-not-migrate)

Query the current ClickHouse docs via context7 `/clickhouse/clickhouse-docs` (alt `/websites/clickhouse`) for exact DDL and engine options.

## When

Migrate a table to ClickHouse when:
- It's a **Tier 2 append-only stream** (event/log/metric) that has grown large.
- Read patterns are dominated by **scan + aggregate over time** (counts, sums, group-by time buckets, top-N), not point lookups-and-edits.
- PostgreSQL is the bottleneck for those analytical reads (or you want them off the OLTP engine).

The middle form (UUID v7 + indexed `created_at` + JsonB payload, append-only) is designed so this move is mechanical, not a redesign.

## What migrates first

**Tier 2 event streams first.** They're already append-only wide tables — the exact shape ClickHouse wants.

**Tier 1 core entities stay in PostgreSQL.** They carry OLTP correctness (FKs, transactions, in-place mutation) that ClickHouse is bad at. If an entity's analytical projection is needed, project it into a ClickHouse table derived from its event stream (a MaterializedView rolling up `order_events` into daily totals), don't move the entity itself.

## Type mapping

| PostgreSQL | ClickHouse | Notes |
|------------|-----------|-------|
| `uuid` | `UUID` | direct |
| `jsonb` | `Object('json')` (old) or the **`JSON`** type (new, recommended) | `JSON` supports paths/indexes |
| `text` / `varchar` | `String` | CH has no varchar length |
| `boolean` | `UInt8` (0/1) | |
| `smallint`/`integer` | `Int16`/`Int32` | |
| `bigint` | `Int64` | |
| `numeric`/`decimal` | `Decimal(P,S)` | |
| `timestamp`/`timestamptz` | `DateTime` / `DateTime64(3)` | use `DateTime64` for sub-second |
| `date` | `Date` / `Date32` | |
| `text[]` (array) | `Array(String)` | |
| `enum` | `Enum8`/`Enum16` or just `LowCardinality(String)` | `LowCardinality(String)` is often simplest |

## ORDER BY and engine choice

ClickHouse `MergeTree` families require an `ORDER BY` that defines on-disk sort + primary index. Design it from your query filters (time range + a filter column).

```sql
CREATE TABLE order_events
(
    id          UUID,
    order_id    UUID,
    type        LowCardinality(String),
    actor_id    String,
    occurred_at DateTime64(3),
    payload     JSON,
    created_at  DateTime64(3)
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(occurred_at)          -- monthly partitions; drop old months cheaply
ORDER BY (occurred_at, id);                  -- time-first; UUID v7 is also time-ordered, doubly good
```

Engine choice:
- **`MergeTree`** — default, for pure append-only streams (our Tier 2 default).
- **`ReplacingMergeTree`** — if your source kept superseded rows and you want CH to dedupe by version on merges (dedup is eventual, force with `FINAL` or `argMax`).
- **`AggregatingMergeTree`** + MaterializedView — for pre-aggregated rollups (daily/hourly totals).

Partition by time (`toYYYYMM`/`toYYYYMMDD`) so dropping old data is an instant partition drop.

## Append-only conversion

Tier 2 tables are **already append-only** in PostgreSQL (no UPDATE/DELETE — that's the discipline from `migration-templates.md`). So conversion is mostly a copy:

1. Export: `lerd db:export -d <db> -o events.sql` (or use `COPY ... TO STDOUT` for CSV/TSV — faster).
2. Map types per the table above.
3. Create the CH table with the right ENGINE/PARTITION BY/ORDER BY.
4. Ingest (below).

If a legacy PG table *did* use UPDATE/DELETE, decide the policy before migrating:
- Keep full history: copy every row version with a version/validity column → `ReplacingMergeTree`.
- Snapshot only: copy the latest state → `MergeTree` (lose history).
- Change-data-capture: stream changes via logical replication / Kafka.

## Ingestion

- **One-time bulk:** `INSERT INTO ... SELECT` from a file, or `clickhouse-client --query "INSERT ..." < data.tsv`. For large loads use the row-binary or native format.
- **Ongoing stream:** Kafka engine → MaterializedView into MergeTree is the idiomatic CH pattern; or HTTP `/insert` from a Laravel job; or a tool like `pg_ch_copy`/Airbyte.
- **Dual-write during transition:** keep writing PG (source of truth) and mirror to CH until you trust the pipeline; then cut analytical reads over to CH.

## Querying from Laravel

ClickHouse speaks SQL over HTTP/native. Options:
- A package like `the-tinder/clickhouse` or `smi2/phpClickHouse` (Laravel-friendly wrapper).
- Or treat CH as a read-only analytics DB and query via `DB::connection('clickhouse')->select(...)` once a connection is configured in `config/database.php`.

Don't try to route Eloquent models to ClickHouse — it's an analytics engine, not an ORM target. Use raw queries / a thin repository for analytics, keep Eloquent on PG for OLTP.

## Do not migrate

- Tier 1 core entities (users, orders-as-entity, accounts) — they need OLTP semantics; keep in PG.
- Small lookup/reference tables — no benefit.
- Anything whose primary access is point-read-and-edit — ClickHouse is the wrong tool.

The architecture's promise: because you used UUID v7 + append-only + JsonB from day one, the decision of *whether* and *when* to migrate each table stays open and cheap. You migrate streams when load demands it, and leave the rest.
