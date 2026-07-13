# Data Architecture — First Principle

## Table of Contents
- [The first principle](#the-first-principle)
- [Constraint A: Tiered modeling](#constraint-a-tiered-modeling)
- [Constraint B: Unified UUID v7 primary keys](#constraint-b-unified-uuid-v7-primary-keys)
- [JsonB usage rules](#jsonb-usage-rules)
- [The bidirectional migration paths](#the-bidirectional-migration-paths)
- [Decision checklist for a new table](#decision-checklist-for-a-new-table)

## The first principle

This project adopts a **wide-table + JsonB middle form** as the default data shape, designed so that any table can migrate **bidirectionally** later:

- **Toward normalized SQL** — promote high-frequency JsonB keys to columns, extract relation tables.
- **Toward ClickHouse** — keep the append-only wide table as-is, use time + UUID for the `ORDER BY`.

PostgreSQL is the starting engine; it is chosen precisely because its JSONB, array, enum, GIN index, and full-text features make this middle form natural, and because it can be left behind for either destination without a rewrite. The goal is to **never be locked in** by a shape that only one engine accepts.

Two hard constraints make this work: **tiered modeling** (A) and **unified UUID v7 keys** (B).

## Constraint A: Tiered modeling

Not every table is built the same way. Classify each table into one of two tiers at design time.

### Tier 1 — Core business entities (normalized)

**Examples:** users, organizations, accounts, orders, subscriptions, products, invoices.

**Modeling:** full relational normalization — foreign-key constraints, JOINs, `references()` in migrations, `softDeletes()` for safe removal, strict column types. These carry the OLTP correctness guarantees (referential integrity, constraints, transactions) that the business depends on.

**Lifecycle:** create / update / soft-delete. Migrations have `up()` and `down()`. Eloquent relations (`hasMany`, `belongsTo`, `morphTo`, …) model the graph.

**Migration target:** stays in PostgreSQL (or another OLTP engine). Not a ClickHouse candidate.

### Tier 2 — Event / log / behavioral streams (append-only wide tables)

**Examples:** audit logs, user activity events, page views, search queries, metric samples, job execution records, state-change history, notifications sent.

**Modeling:** **append-only wide table.** One row per event. No `UPDATE`, no `DELETE` (corrections are new rows that supersede). A stable set of "hot" query columns extracted to real columns; the variable/restless payload lives in a JsonB column. Time is first-class and indexed.

**Lifecycle:** insert + read. Soft-deletes are **not** used (they imply UPDATE). A "correction" is an appended row with a reference to the superseded row, not an edit. This discipline is what makes the ClickHouse migration nearly free — ClickHouse mutations (`UPDATE`/`DELETE`) are expensive and discouraged.

**Migration target:** **ClickHouse first candidate.** Design it now so the move is mechanical (see [clickhouse-migration.md](clickhouse-migration.md)).

### How to classify

Ask:
1. Does it have identity that must be referenced by other tables via FK? → Tier 1.
2. Is it mutated in place by the business (edit a user's email, change an order status)? → Tier 1.
3. Is it a stream of facts that are true at a point in time and never revised in place? → Tier 2.
4. Is the read pattern overwhelmingly "aggregate/scan over time" rather than "fetch one by key and edit"? → Tier 2.

Ambiguous? Default to Tier 1 (normalized) and extract an event stream alongside it. E.g. an `orders` table (Tier 1) plus an `order_events` append-only stream (Tier 2) recording every status transition — you keep OLTP correctness *and* get an analytics-ready stream.

## Constraint B: Unified UUID v7 primary keys

**All new tables use UUID v7 as the primary key, regardless of tier.** This is non-negotiable for new tables; it's what makes the bidirectional migration possible without a painful key-type change.

### Why UUID v7

- **Time-ordered** (timestamp prefix) → B-tree index locality in PostgreSQL stays good (no random-insert fragmentation that plain v4 UUIDs cause), and it doubles as a ClickHouse `ORDER BY` component.
- **Globally unique** → safe across shards/services, no central counter, no merge conflicts when combining data from multiple sources.
- **One type everywhere** → migrating a table to ClickHouse never hits a "bigint vs uuid" key clash; joins across migrated and unmigrated tables keep working during a phased migration.

### How (Laravel)

Laravel (11.x+/13.x) ships a **`HasUuids`** trait that auto-generates a UUIDv7 for the primary key. Use it instead of hand-writing `$keyType`/`$incrementing`/a `creating` listener.

Migration:
```php
$table->uuid('id')->primary();   // NOT $table->id()
$table->foreignUuid('user_id')->constrained()->cascadeOnDelete();  // FKs are uuid too
```

Model:
```php
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;

class Order extends Model
{
    use HasUuids;   // auto-fills id with Str::uuid7() on create
    // $keyType='string' and $incrementing=false are handled by the trait
}
```

For raw generation: `use Illuminate\Support\Str; (string) Str::uuid7();` (optionally `Str::uuid7(time: $someDateTime)`).

Factories: omit `id` from the definition — the trait fills it. Only set FKs: `'user_id' => User::factory()`.

See `laravel-conventions`'s [models-and-casts.md](../../laravel-conventions/references/models-and-casts.md) for the full base-class pattern.

### On existing tables

If a legacy table uses bigint auto-increment, do **not** force-convert mid-stream. Leave it, and if it ever migrates to ClickHouse, add a UUID v7 column then. The UUID-v7 rule applies to **new** tables.

## JsonB usage rules

JsonB is the "restless data" home. The discipline is: **stable + frequently-queried → real column; variable / semi-structured / rare-query → JsonB.**

### Do put in JsonB
- Variable, extensible payloads: `context`, `metadata`, `payload`, `settings`, `attributes`, `request_snapshot`.
- Nested objects/arrays whose shape evolves (feature flags, integration configs, webhook bodies).
- Data you'd otherwise shove into EAV or a dozen nullable columns.

### Do NOT put in JsonB
- Anything you `WHERE`/`ORDER BY`/`JOIN` on frequently — promote it to a real indexed column.
- Foreign keys (FKs must be real columns).
- Fields needing a CHECK or uniqueness constraint (enforce on real columns; JsonB constraints are awkward).

### Indexing JsonB (see [postgres-features.md](postgres-features.md))
- GIN index on the whole column: `CREATE INDEX ON items USING GIN (meta);` — supports `@>`, `?`, key existence, `jsonb_path_query`.
- Expression index for a hot key: `CREATE INDEX ON items ((meta->>'category'));` — supports `WHERE meta->>'category' = 'x'` as a normal B-tree.
- Prefer the **expression index** when you query one or two known keys; use the GIN when query patterns are open-ended.

### Eloquent casts
```php
protected $casts = [
    'meta'     => 'array',                 // quick: json ↔ array
    'address'  => Address::class,          // value-object cast (DTO with casts())
    'tags'     => 'array',                 // json array of scalars
];
```
For nested domain objects, write a custom cast implementing `Cast::get`/`set`. See `laravel-conventions`'s models-and-casts.md.

## The bidirectional migration paths

The middle form is valuable precisely because both exits are open. Plan the shape so neither is closed.

### → Normalized SQL (refactor in place, stay in PostgreSQL)

Trigger: a JsonB key becomes hot — queried/sorted/grouped on constantly.

1. Add the real column (nullable first): `$table->string('category')->nullable()->after('meta');`
2. Backfill: `UPDATE items SET category = meta->>'category';`
3. Add index, add constraints if needed.
4. Stop writing the key to `meta` (update the cast/value-object).
5. Optionally drop the key from `meta`.

Trigger: a table grows relations — extract.

1. Create the child table (UUID v7 key, FK back).
2. Migrate data, update reads/writes.
3. Drop the denormalized copy.

This path **stays in PostgreSQL** and improves OLTP shape. No engine change.

### → ClickHouse (engine change, for Tier 2 streams first)

Trigger: a Tier 2 event table grows large; analytical queries (scan + aggregate over time) dominate; PG is the bottleneck.

1. The wide table is already append-only with UUID v7 + `created_at` → map directly to a ClickHouse `MergeTree` (or `ReplacingMergeTree` if you kept superseded rows) with `ORDER BY (created_at, id)`.
2. Map types: `uuid→UUID`, `jsonb→Object('json')` or the new JSON type, `timestamptz→DateTime64(3)`, `text→String`, `bool→UInt8`, numeric → `Int*`/`UInt*`/`Decimal`.
3. Ingest: stream inserts (Kafka engine / HTTP `/insert`) or batch copy. Keep the PG table as source-of-truth during transition if you still mutate it.
4. Tier 1 entities migrate **last**, if ever — they stay in PG for OLTP. ClickHouse holds the streams and aggregates.

Full type mapping, ORDER BY design, and append-only conversion in [clickhouse-migration.md](clickhouse-migration.md). Query ClickHouse docs via context7 `/clickhouse/clickhouse-docs`.

## Decision checklist for a new table

Before writing a migration, answer:

1. **Tier?** Entity (normalized) or event stream (append-only wide)?
2. **Primary key?** `$table->uuid('id')->primary();` (always, for new tables).
3. **Hot query columns?** Promote to real columns + indexes.
4. **Variable payload?** JsonB column + GIN and/or expression index.
5. **Time?** Tier 2 tables must have `created_at` indexed (and consider it for the future ORDER BY).
6. **Lifecycle?** Tier 1 → `timestamps()` + `softDeletes()` + up/down migrations. Tier 2 → `timestamps()` (use `created_at`), **no** softDeletes, immutable rows.
7. **FKs?** Tier 1 uses `foreignUuid(...)->constrained()`. Tier 2 may denormalize a snapshot of the referenced entity (append-only can't rely on FK for event history).
8. **Reversible?** Every `up()` needs a `down()` that truly reverses (see lerdrail-workflow quality gates).

Answering these up front is the whole point of this skill — it prevents the shape decisions that lock you in.
