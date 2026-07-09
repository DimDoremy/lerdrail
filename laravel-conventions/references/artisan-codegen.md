# Artisan Code Generation

## Table of Contents
- [Why generate, then edit](#why-generate-then-edit)
- [Run via Lerd](#run-via-lerd)
- [Common make commands](#common-make-commands)
- [Post-generation data-architecture edits](#post-generation-data-architecture-edits)
- [Listing commands](#listing-commands)

## Why generate, then edit

`lerd artisan make:*` produces consistent stubs faster and more reliably than hand-writing. The one mandatory post-step: **enforce the project's data-architecture rules** on anything it generates, because the default stubs use `$table->id()` (bigint auto-increment), not the project's UUID v7.

## Run via Lerd

Always run artisan through Lerd so it executes in the project's correct PHP container/version:

```bash
lerd artisan make:model Order -mfsc
```

(Not `php artisan ...` from the host, which may hit the wrong PHP.) MCP equivalent: `exec` tool with `action: "artisan"`.

## Common make commands

| Command | Generates |
|---------|-----------|
| `make:model Order -mfsc` | Model + Migration + Factory + Seeder + Controller |
| `make:model Order -a` | All: model, factory, migration, seeder, controller, form request, policy |
| `make:controller OrderController --model=Order --requests` | Resource controller + Store/Update form requests |
| `make:controller OrderController --invokable` | Single-action controller |
| `make:request StoreOrderRequest` | Form Request (validation + authorize) |
| `make:resource OrderResource` | API Resource transformer |
| `make:resource OrderCollection` | Resource collection (for list shaping) |
| `make:policy OrderPolicy --model=Order` | Authorization policy |
| `make:job ProcessOrder` | Queueable job |
| `make:event OrderPlaced` | Event class |
| `make:listener PlaceOrder --event=OrderPlaced` | Listener (optionally queued) |
| `make:notification OrderShipped` | Notification |
| `make:observer OrderObserver` | Model lifecycle observer |
| `make:middleware EnsureAuditContext` | HTTP middleware |
| `make:command SendInvoices` | Artisan console command (`signature`/`handle`) |
| `make:migration create_orders_table` | Empty migration |
| `make:seeder OrderSeeder` | Seeder |
| `make:test OrderTest` (Pest) | Pest test file |
| `make:cast AddressCast` | Custom attribute cast |
| `make:enum OrderStatus` (if on a supported Laravel version) | Backed enum |

Use `lerd artisan list make` to see all available makers (or Boost `commands` for the live set, including package-registered ones).

## Post-generation data-architecture edits

Every generated **migration** must be fixed for the project's UUID v7 rule. Before running it:

```diff
- $table->id();
+ $table->uuid('id')->primary();
- $table->unsignedBigInteger('user_id');
- $table->foreign('user_id')->references('id')->on('users');
+ $table->foreignUuid('user_id')->constrained()->cascadeOnDelete();
```
- Add `$table->jsonb('meta')->nullable();` for restless payloads (don't invent columns for every variable field).
- Tier 1 (core entity): keep `$table->timestamps(); $table->softDeletes();`
- Tier 2 (event stream): `$table->timestamps();` but `$table->timestampTz('created_at')` indexed and **no** `softDeletes`, no `updated_at` use.
- Add indexes on FKs and time columns; expression/GIN indexes on JsonB hot keys.
- Write a real `down()` that reverses (drop indexes → drop table / drop type).

Every generated **model**:
```php
use Illuminate\Database\Eloquent\Concerns\HasUuids;
class Order extends Model   // App\Models\Model base already uses HasUuids
{
    use SoftDeletes;        // Tier 1 only
    protected $casts = [
        'status' => OrderStatus::class,
        'meta'   => 'array',
    ];
}
```
Omit `'id'` from factories.

See [models-and-casts.md](models-and-casts.md) and [postgres-valkey/migration-templates.md](../../postgres-valkey/references/migration-templates.md).

## Listing commands

For the live, app-aware set (including commands registered by installed packages like Boost, Horizon, etc.), use **Boost's `artisan commands` tool** — it introspects the running app, which is more complete than the static `lerd artisan list` because it reflects the actual container.

Run a command: `lerd artisan <name>` (or MCP `exec` `artisan`). For project convenience commands registered in `.lerd.yaml`, `lerd run <name>` respects project overrides.
