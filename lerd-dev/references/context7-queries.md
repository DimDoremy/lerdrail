# context7 Library IDs & Query Templates

Pre-resolved library IDs so you don't re-resolve each session. Use `mcp__context7__query-docs` with the `libraryId` below.

## Laravel — Boost first, context7 fallback

Boost's Documentation Search is tuned to the Laravel version actually installed in the project, so it's more accurate than context7 for Laravel APIs. **Try Boost first.** Fall back to context7 only when Boost doesn't surface what you need.

- **`/laravel/docs`** — official docs (versions: 10.x, 11.x, 13.x branches). Pick the branch matching the project.
- Alternative: `/websites/laravel_12_x` (Laravel 12.x site).

Useful queries:
- "artisan make commands list" (codegen — see `laravel-conventions`)
- "HasUuids trait UUID primary key" (data architecture — see `postgres-valkey`)
- "eloquent relationships hasMany belongsTo morphMany"
- "migrations available column types jsonb uuid"
- "pest testing feature test assertStatus RefreshDatabase"
- "inertia render props sharing"

## React

- **`/reactjs/react.dev`** — official docs (benchmark 90.67).
- Queries: "useState useEffect", "component composition patterns", "custom hooks", "forms controlled inputs".

## Tailwind CSS

- **`/tailwindlabs/tailwindcss.com`** — official docs (v4 focus).
- Queries: "responsive design breakpoints", "dark mode", "customizing theme tokens", "common utility patterns", "grid flexbox".

## Inertia.js

- **`/inertiajs/docs`** — main docs (v2/v3). Use with:
- **`/inertiajs/inertia-laravel`** — Laravel adapter specifics.
- Queries: "React useForm helper", "server-side rendering setup", "shared data Inertia::share", "partial reloads only", "prefetching", "optimistic updates", "links prefetch".
- React `useForm` returns `{ data, setData, post/get/put/delete, processing, errors, wasSuccessful, recentlySuccessful, progress, reset, clearErrors, isDirty, defaults, optimistic }`.

## Pest

- **`/pestphp/docs`** — Pest framework docs.
- **`/pestphp/pest-plugin-browser`** — browser testing plugin (Playwright-based; the project's chosen browser-testing path, see `dev-workflow`).
- Queries: "dataset provider", "beforeEach afterEach", "expect API", "architecture testing", "browser testing setup chromium".

## PostgreSQL

- **`/websites/postgresql_current`** — current docs (highest snippet count for "current").
- Queries: "JSONB operators indexing GIN", "array column type operators", "CREATE TYPE enum", "full text search tsvector tsquery GIN", "expression index", "partial index WHERE", "CTE WITH RECURSIVE", "window functions OVER PARTITION".

## Valkey

- **`/websites/valkey_io`** — Valkey docs.
- Note: Valkey is wire-compatible with Redis. For Laravel usage the Redis docs apply identically (`REDIS_HOST=lerd-redis`). Valkey-specific queries: "Valkey vs Redis differences", "Valkey commands", "Valkey cluster".

## ClickHouse

- **`/clickhouse/clickhouse-docs`** — official docs.
- Alternative: `/websites/clickhouse` (more snippets).
- Queries (for the PG→ClickHouse migration path, see `postgres-valkey`): "MergeTree engine ORDER BY", "UUID type", "JSON Object type", "DateTime64", "ReplacingMergeTree deduplication", " Kafka engine ingestion", "MaterializedView", "dictionary PostgreSQL source".

## Querying tips

- One concept per `query-docs` call. "JSONB indexing" and "full-text search" are separate calls.
- If a query is too broad, refine to a specific API ("GIN index on jsonb path expression") rather than the topic ("jsonb").
- context7 returns code snippets sourced from real docs — prefer them over pretraining knowledge for exact signatures.
