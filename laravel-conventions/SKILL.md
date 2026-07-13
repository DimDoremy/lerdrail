---
name: laravel-conventions
description: Use when writing Laravel backend code—artisan code generation, Eloquent models/migrations/factories/policies, form requests/resources, jobs/events/notifications, routing, and Pest tests—in an Inertia.js + React project. Enforces the project's data-architecture rules (UUID v7 primary keys via the HasUuids trait, JsonB casts for semi-structured data). Defers live introspection (schema, routes, config, logs, tinker, doc lookup) to Laravel Boost MCP, and code execution to `lerd artisan`/`lerd test`. Covers the controller→Inertia props→React page data flow.
---

# Laravel Conventions

## How this skill relates to other layers

- **Live app introspection** (what's the schema, what routes exist, what does this config resolve to, read the log, run ad-hoc tinker, list artisan commands, look up a Laravel API) → **defer to Boost MCP**. Don't re-derive from files or pretraining.
- **Running artisan / tests / migrations** → **Lerd** (`lerd artisan`, `lerd test`, `lerd artisan migrate`). There is no `lerd exec`.
- **Schema/data-architecture decisions** (UUID v7, JsonB, tiered modeling, indexes) → the [first principle in `postgres-valkey`](../postgres-valkey/references/data-architecture.md). This skill implements it on the model/migration side.
- **Laravel docs** → Boost Documentation Search first; context7 `/laravel/docs` fallback.
- **Front-end half of Inertia** (React pages, Tailwind, Vite/SSR) → `react-tailwind`.

## Data-architecture rules this skill enforces

Every model and migration obeys the project's [first data-architecture principle](../postgres-valkey/references/data-architecture.md):

1. **UUID v7 primary keys on all new tables** — via the `HasUuids` trait (auto-generates `Str::uuid7()`). Migration uses `$table->uuid('id')->primary()`, never `$table->id()`. FKs are `foreignUuid(...)->constrained()`.
2. **JsonB for restless payloads** — `$casts = ['meta' => 'array']` or a custom value-object cast; index hot keys with expression indexes.
3. **Tiered modeling** — core entities are normalized (`softDeletes`, FKs, JOINs); event/log streams are append-only (no `softDeletes`, no UPDATE/DELETE).

Full model base pattern, casts, factories in [references/models-and-casts.md](references/models-and-casts.md). Migration templates in [postgres-valkey/migration-templates.md](../postgres-valkey/references/migration-templates.md).

## Code generation: artisan first

Prefer `lerd artisan make:*` over hand-writing boilerplate — it produces consistent stubs. Run via Lerd so it executes in the project's PHP container:

```bash
lerd artisan make:model Order -mfsc      # model + migration + factory + seeder + controller
lerd artisan make:request StoreOrderRequest
lerd artisan make:resource OrderResource
lerd artisan make:policy OrderPolicy --model=Order
lerd artisan make:job ProcessOrder
lerd artisan make:listener OrderPlaced --event=OrderPlaced
lerd artisan make:notification OrderShipped
lerd artisan make:controller OrderController --model=Order --requests
```

After generation, edit to enforce the data rules: swap `$table->id()` → `$table->uuid('id')->primary()`, add `use HasUuids` to the model, add JsonB casts. Full command catalog and post-generation edits in [references/artisan-codegen.md](references/artisan-codegen.md).

> **Listing available artisan commands** → Boost `artisan commands` tool (it introspects the running app, including registered commands from packages).

## Layering: Request → Controller → Resource → Inertia

- **Form Request** for validation + authorization (`rules()`, `authorize()`). Keeps controllers thin.
- **Controller** orchestrates: validate (via request), act on models, dispatch jobs/events, then render.
- **API Resource** to shape the outgoing payload — don't leak Eloquent models straight to Inertia; transform through a resource so the React contract is explicit and stable.
- **Inertia response** — `Inertia::render('Orders/Show', $props)`.

The controller→props→page flow, shared data, partial reloads, and form handling in [references/inertia-react.md](references/inertia-react.md).

## Models & relationships

- `HasUuids` trait on every model.
- Relations declared as methods (`hasMany`, `belongsTo`, `morphTo`, `morphMany`).
- Query scopes for reusable filters (`scopePublished`).
- Accessors/mutators via `Attribute::make`.
- Casts: backed enums, datetime, json/array for JsonB, custom value-object casts.

Details in [references/models-and-casts.md](references/models-and-casts.md).

## Testing (php / http / console)

Three of the four test gates live here; browser testing is in `lerdrail-workflow`.

- **Pest** is the project's test framework (`lerd artisan test` / `lerd pest`).
- **HTTP tests** (`$this->get/post`, `assertStatus`, `assertJson`, `assertSessionHas`) for endpoints.
- **Console tests** (`$this->artisan(...)`, `Artisan::call`) for commands.
- Data: `RefreshDatabase` trait (migrates the `_testing` DB), factories for entities, `DatabaseSeeder` for shared fixtures, Mockery for collaborators.
- Run: `lerd test --filter=OrderTest` or `lerd pest`.

Full conventions and patterns in [references/testing.md](references/testing.md). The test *gates* (what must pass, when) are defined in `lerdrail-workflow`'s [testing-gates.md](../lerdrail-workflow/references/testing-gates.md).

## Quick reference

| Want to… | Do |
|----------|----|
| Generate a model + migration + factory + controller | `lerd artisan make:model Order -mfsc` |
| Apply UUID v7 to a model | `use HasUuids;` (trait sets keyType/incrementing, fills id) |
| Cast a JsonB column | `protected $casts = ['meta' => 'array'];` |
| Validate input | Form Request `rules()` |
| Shape API output | API Resource `->toArray($request)` |
| Render an Inertia page | `Inertia::render('Orders/Show', ['order' => OrderResource::make($order)])` |
| Run tests | `lerd test` (or `lerd pest --filter=X`) |
| Inspect the live schema/routes | Boost `schema` / `route inspector` |
| Look up a Laravel API | Boost doc search → context7 `/laravel/docs` |

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Hand-writing a model instead of `make:model` | Use artisan, then edit for data rules |
| `$table->id()` after generation | Replace with `$table->uuid('id')->primary()` + `HasUuids` |
| Leaking Eloquent to Inertia | Wrap in an API Resource |
| Putting validation in controllers | Move to a Form Request |
| `$ php artisan test` from host | Use `lerd test` (correct PHP container/version) |
| Re-deriving schema from migration files | Use Boost `schema` for ground truth |
