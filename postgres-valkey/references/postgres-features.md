# PostgreSQL Features

## Table of Contents
- [JsonB](#jsonb)
- [Arrays](#arrays)
- [Native enums](#native-enums)
- [Full-text search](#full-text-search)
- [Indexes](#indexes)
- [CTEs and window functions](#ctes-and-window-functions)
- [Laravel raw patterns](#laravel-raw-patterns)

Query the current PG docs via context7 `/websites/postgresql_current` for exact operator signatures.

## JsonB

The workhorse for the "restless payload" part of the middle form. Operators you'll use most:

| Operator | Meaning |
|----------|---------|
| `->` | get value as jsonb (`meta->'address'`) |
| `->>` | get value as text (`meta->>'category'`) |
| `#>>` | path as text (`meta #>> '{address,city}'`) |
| `@>` | contains (`meta @> '{"role":"admin"}'`) — GIN-accelerated |
| `?` | key exists (`meta ? 'feature_flags'`) |
| `?|` / `?&` | any / all keys exist |
| `jsonb_path_query` | SQL/JSON path (complex extraction) |

Laravel migration:
```php
$table->jsonb('meta')->nullable();
$table->jsonb('context')->nullable();
```

Laravel query builder:
```php
// contains
Item::where('meta', '@>', ['role' => 'admin'])->get();
// key exists
Item::whereJsonContains('meta->tags', 'urgent')->get();
// text path (use whereRaw for operators not in builder)
Item::whereRaw("meta->>'category' = ?", ['books'])->get();
```

Eloquent cast:
```php
protected $casts = ['meta' => 'array'];
```

### Indexing JsonB

- **GIN on the whole column** — supports `@>`, `?`, path queries:
  ```php
  // in a migration
  DB::statement('CREATE INDEX items_meta_gin ON items USING GIN (meta)');
  ```
- **Expression index on one hot key** — supports `meta->>'category' = 'x'` as a normal B-tree:
  ```php
  DB::statement("CREATE INDEX items_category_idx ON items ((meta->>'category'))");
  ```
  Prefer the expression index when you query one or two known keys; use GIN when query patterns are open-ended. GIN indexes are larger and slower to update.

## Arrays

For simple multi-value columns where you don't need FK integrity (tags, role lists, coordinate pairs).

```php
$table->string('tags')->array();      // text[]
$table->integer('scores')->array();   // int[]
```
```php
// query: contains
Item::whereRaw('tags @> ARRAY[?]', ['urgent'])->get();
Item::whereJsonContains('tags', 'urgent')->get(); // builder helper, works for jsonb & arrays on PG
// any match
Item::whereRaw('? = ANY(tags)', ['urgent'])->get();
// array length
Item::whereRaw('array_length(tags, 1) > 3')->get();
```
Cast: `'tags' => 'array'`. Don't use arrays for things that need to be JOINed/FK'd — use a relation table.

## Native enums

For a fixed vocabulary that the DB should enforce. Two routes:

**Laravel `enum` (cheap, varchar + CHECK-ish):**
```php
$table->enum('status', ['draft', 'published', 'archived']);
```

**Real PG enum type (faster, stricter, shown in `\d`):**
```php
// migration up
DB::statement("CREATE TYPE order_status AS ENUM ('draft','submitted','paid','shipped')");
$table->string('status'); // then bind via raw, or use a package like spitouthollows enums

// migration down — drop type AFTER the column
DB::statement("DROP TYPE order_status");
```
For PHP-side type safety, pair with a backed enum:
```php
enum OrderStatus: string { case Draft='draft'; case Paid='paid'; /* ... */ }
// model
protected $casts = ['status' => OrderStatus::class];
```

## Full-text search

Before reaching for Meilisearch, check whether PG full-text is enough (it usually is up to low millions of rows).

```php
// migration: generated tsvector column + GIN
DB::statement("ALTER TABLE articles ADD COLUMN search_vector tsvector
    GENERATED ALWAYS AS (to_tsvector('english', coalesce(title,'') || ' ' || coalesce(body,'')))");
DB::statement("CREATE INDEX articles_search_idx ON articles USING GIN (search_vector)");
```
```php
// query
Article::whereRaw("search_vector @@ plainto_tsquery('english', ?)", [$term])->get();
// ranked
Article::selectRaw("*, ts_rank(search_vector, plainto_tsquery('english', ?)) AS rank", [$term])
    ->whereRaw("search_vector @@ plainto_tsquery('english', ?)", [$term])
    ->orderByDesc('rank')->get();
```

## Indexes

| Type | When |
|------|------|
| B-tree (default) | equality / range on a column |
| GIN | JsonB (`@>`/`?`), arrays, full-text (`tsvector`) |
| GiST | geometric, exclusion constraints, some tsvector configs |
| BRIN | huge, naturally-ordered tables (time-series) — tiny index, great for Tier 2 streams |
| Hash | equality only (rare) |

**Partial index** (index only matching rows — smaller, faster):
```php
DB::statement("CREATE INDEX orders_unshipped_idx ON orders (created_at) WHERE shipped_at IS NULL");
```
**Expression index** (index a computed value):
```php
DB::statement("CREATE INDEX users_lower_email_idx ON users ((lower(email)))");
DB::statement("CREATE INDEX items_category_idx ON items ((meta->>'category'))");
```

## CTEs and window functions

For analytics queries that would otherwise need multiple round-trips.

```php
// CTE: e.g. per-user latest order
$latest = DB::table('orders')
    ->select('user_id', DB::raw('MAX(created_at) as latest'))
    ->groupBy('user_id');

$rows = DB::table('orders')
    ->joinSub($latest, 'l', fn($j) => $j->on('orders.user_id','l.user_id')->on('orders.created_at','l.latest'))
    ->get();

// raw WITH for recursion (e.g. org hierarchy)
DB::statement("WITH RECURSIVE ... ");
```

Window functions (running totals, ranks, row numbers):
```php
$rows = DB::table('orders')
    ->select('id', 'user_id', 'total', DB::raw("ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at) AS rn"))
    ->get();
```

## Laravel raw patterns

When the builder can't express a PG-specific operator, drop to `whereRaw`/`selectRaw`/`DB::statement` — that's normal and expected. Keep the operator in SQL, bind values as parameters (`?`), never string-interpolate values.

For DDL that Laravel's schema builder can't express (CREATE TYPE, generated columns, custom indexes, triggers), use `DB::statement(...)` inside the migration's `up()`/`down()`. Remember `down()` must reverse everything in reverse order (drop indexes before columns/types).
