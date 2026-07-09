# Migration Templates

## Table of Contents
- [Shared conventions](#shared-conventions)
- [Tier 1 — core entity](#tier-1--core-entity)
- [Tier 2 — event stream (append-only)](#tier-2--event-stream-append-only)
- [Factories](#factories)
- [Reversibility](#reversibility)

Both templates follow the [data-architecture first principle](data-architecture.md): UUID v7 primary keys always, tiered shape, JsonB for restless payloads. Copy these and adapt.

## Shared conventions

- Primary key: `$table->uuid('id')->primary();` — never `$table->id()`.
- Foreign keys: `$table->foreignUuid('user_id')->constrained()->cascadeOnDelete();` (uuid to match).
- Timestamps: `$table->timestamps();` (`created_at`, `updated_at`).
- Index foreign keys you query by: `$table->index('user_id');`
- Keep migrations **reversible** — every `up()` has a `down()` that truly reverses.

Generate with: `lerd artisan make:migration create_orders_table`.

## Tier 1 — core entity

Normalized: FKs, JOINs, soft-deletes, strict types. Lifecycle = create/update/soft-delete. **Stays in PostgreSQL.**

```php
// xxxx_xx_xx_xxxxxx_create_orders_table.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::create('orders', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->foreignUuid('user_id')->constrained()->cascadeOnDelete();
            $table->string('number')->unique();          // a stable business key
            $table->string('status');                    // pair with a backed enum cast
            $table->unsignedInteger('amount_cents');
            $table->string('currency', 3)->default('USD');

            // restless / extensible payload — JsonB
            $table->jsonb('meta')->nullable();           // notes, flags, integration refs

            $table->timestamps();
            $table->softDeletes();

            $table->index('user_id');
            $table->index('status');
            $table->index('created_at');
        });

        // index a hot JsonB key as an expression (see postgres-features.md)
        DB::statement("CREATE INDEX orders_channel_idx ON orders ((meta->>'channel'))");
    }

    public function down(): void
    {
        DB::statement("DROP INDEX IF EXISTS orders_channel_idx");
        Schema::dropIfExists('orders');
    }
};
```

Model:
```php
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Order extends Model
{
    use HasUuids, SoftDeletes;

    protected $casts = [
        'status' => OrderStatus::class,   // backed enum
        'meta'   => 'array',
    ];

    public function user() { return $this->belongsTo(User::class); }
    public function items() { return $this->hasMany(OrderItem::class); }
}
```

## Tier 2 — event stream (append-only)

Append-only wide table. UUID v7 + indexed `created_at` (future ClickHouse ORDER BY), JsonB payload, **no softDeletes**, **no UPDATE/DELETE** — corrections are appended rows. **ClickHouse migration candidate.**

```php
// xxxx_xx_xx_xxxxxx_create_order_events_table.php
return new class extends Migration {
    public function up(): void
    {
        Schema::create('order_events', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->foreignUuid('order_id')->constrained()->cascadeOnDelete();

            // hot query columns — promoted from the payload because we filter/group on them
            $table->string('type');                      // e.g. 'status_changed'
            $table->string('actor_id')->nullable();      // who/what caused it
            $table->timestampTz('occurred_at')->useCurrent();

            // the restless payload — everything variable lives here
            $table->jsonb('payload')->nullable();        // from/to values, snapshot, context

            $table->timestampTz('created_at')->useCurrent();
            // NOTE: no updated_at, no softDeletes — append-only.

            $table->index(['order_id', 'occurred_at']);
            $table->index('type');
            $table->index('occurred_at');               // critical for time scans / CH ORDER BY
        });

        // GIN for open-ended payload queries
        DB::statement("CREATE INDEX order_events_payload_gin ON order_events USING GIN (payload)");
    }

    public function down(): void
    {
        DB::statement("DROP INDEX IF EXISTS order_events_payload_gin");
        Schema::dropIfExists('order_events');
    }
};
```

Model:
```php
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;

class OrderEvent extends Model
{
    use HasUuids;

    public const UPDATED_AT = null;            // append-only: no updated_at
    protected $casts = ['payload' => 'array'];

    // Provide an append API instead of an update API:
    public static function record(Order $order, string $type, array $payload = [], ?string $actorId = null): self
    {
        return static::create([
            'order_id'   => $order->id,
            'type'       => $type,
            'actor_id'   => $actorId,
            'occurred_at' => now(),
            'payload'    => $payload,
        ]);
    }
}
```

**Discipline:** never call `$event->update(...)` or `$event->delete()` on a Tier 2 row. To "correct" an event, append a new row whose `payload` references the superseded event id. This is what makes the ClickHouse move mechanical — see [clickhouse-migration.md](clickhouse-migration.md).

## Factories

```php
// database/factories/OrderFactory.php
public function definition(): array
{
    return [
        // 'id' omitted — HasUuids fills it
        'user_id'       => User::factory(),
        'number'        => 'ORD-' . fake()->unique()->numerify('######'),
        'status'        => OrderStatus::Draft,
        'amount_cents'  => fake()->numberBetween(100, 100000),
        'currency'      => 'USD',
        'meta'          => ['channel' => fake()->randomElement(['web','api','mobile'])],
    ];
}
```

For JsonB, prefer realistic nested structures in the factory so tests exercise the real shape.

## Reversibility

Every `up()` needs a `down()`. For DDL added via `DB::statement` (indexes, types, generated columns), the `down()` must drop them — and in the right order (drop indexes before columns/types they depend on). dev-workflow's quality gate checks `lerd artisan migrate:status` and that `migrate:rollback` works; verify locally with a snapshot: `lerd db:snapshot pre-migration` before, `lerd db:restore pre-migration` to test rollback.
