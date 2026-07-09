# Models & Casts

## Table of Contents
- [UUID v7 via HasUuids](#uuid-v7-via-hasuuids)
- [A model base class](#a-model-base-class)
- [Relationships](#relationships)
- [Query scopes](#query-scopes)
- [Accessors & mutators](#accessors--mutators)
- [Casts (enum, json, datetime, value-objects)](#casts-enum-json-datetime-value-objects)
- [Factories](#factories)

Implements the [data-architecture first principle](../../postgres-valkey/references/data-architecture.md) on the Eloquent side. Query Laravel docs via Boost Documentation Search or context7 `/laravel/docs`.

## UUID v7 via HasUuids

Laravel (11.x+/13.x) ships the **`HasUuids`** concern. It sets `$keyType='string'`, `$incrementing=false`, and auto-fills the primary key with `Str::uuid7()` on `creating`. You do **not** hand-write those properties or a `creating` listener.

```php
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;

class Order extends Model
{
    use HasUuids;
    // id is auto-filled; $keyType and $incrementing handled by the trait
}

$order = Order::create(['number' => 'ORD-1']);
$order->id; // "018f2b5c-6a7f-7b12-9d6f-2f8a4e0c9c11" (UUIDv7, time-ordered)
```

For raw generation: `(string) Illuminate\Support\Str::uuid7();` — optionally `Str::uuid7(time: $dateTime)` to influence ordering.

**Migrations must match** — `$table->uuid('id')->primary()`, FKs `$table->foreignUuid('user_id')->constrained()`. See [postgres-valkey/migration-templates.md](../../postgres-valkey/references/migration-templates.md).

## A model base class

For project-wide conventions (HasUuids + common casts + soft-delete opt-in), define a base model:

```php
// app/Models/Model.php
namespace App\Models;

use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model as EloquentModel;

abstract class Model extends EloquentModel
{
    use HasUuids;

    public $incrementing = false;
    protected $keyType = 'string';   // belt-and-suspenders; HasUuids sets this too
}
```

Models extend `App\Models\Model`:
```php
class Order extends Model
{
    use SoftDeletes;            // only for Tier 1 (core entities)
    protected $casts = [
        'status' => OrderStatus::class,
        'meta'   => 'array',
    ];
}
```

For **Tier 2 (append-only)** models: do NOT add `SoftDeletes`; set `public const UPDATED_AT = null;`. See [migration-templates.md](../../postgres-valkey/references/migration-templates.md#tier-2--event-stream-append-only).

## Relationships

Declared as methods; use the typed return for IDE/Static-analysis.

```php
public function user()   { return $this->belongsTo(User::class); }
public function items()  { return $this->hasMany(OrderItem::class); }
public function invoice() { return $this->hasOne(Invoice::class); }
public function tags()   { return $this->morphToMany(Tag::class, 'taggable'); }
public function events() { return $this->hasMany(OrderEvent::class); } // Tier 2 stream
```

- Default to **eager loading** (`with`) to avoid N+1 — Lerd's query viewer will flag N+1; use it.
- Use `loadCount` / `withCount` for counts instead of fetching relations just to count.
- Polymorphic relations (`morphTo`/`morphMany`) for shared stream-like tables (activity log → `subject`).

## Query scopes

```php
public function scopePublished(Builder $q): Builder
{
    return $q->whereNotNull('published_at');
}
// usage
Order::published()->where('user_id', $id)->get();
```
Keep scopes composable and parameterizable; don't hide ordering/side-effects in them.

## Accessors & mutators

Use the `Attribute` fluent API (not the old `getXAttribute` methods):

```php
use Illuminate\Database\Eloquent\Casts\Attribute;

protected function amount(): Attribute
{
    return Attribute::make(
        get: fn (int $cents) => new Money($cents, $this->currency),
        set: fn (Money $m) => [$m->amountInCents(), 'currency' => $m->currency],
    );
}
```

## Casts (enum, json, datetime, value-objects)

```php
// Backed enum → stored as its value
enum OrderStatus: string { case Draft='draft'; case Submitted='submitted'; case Paid='paid'; }
protected $casts = ['status' => OrderStatus::class];

// JsonB / arrays
protected $casts = ['meta' => 'array', 'tags' => 'array'];

// datetime
protected $casts = ['published_at' => 'datetime', 'shipped_at' => 'immutable_datetime'];

// Custom value-object cast (for a nested JsonB object)
class Address implements CastsAttributes {
    public function get($model, string $key, $value, array $attributes): ?AddressVO {
        return $value ? AddressVO::fromArray(json_decode($value, true)) : null;
    }
    public function set($model, string $key, $value, array $attributes): ?string {
        return $value ? json_encode($value->toArray()) : null;
    }
}
protected $casts = ['address' => Address::class];
```

Prefer a **custom cast** over `array` when the JsonB value has a defined shape (a value-object/DTO) — it centralizes invariants and gives typed access.

## Factories

```php
class OrderFactory extends Factory
{
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
            'published_at'  => null,
        ];
    }
}
```

- Never set `'id'` in a definition — the trait fills it.
- Use realistic nested JsonB in factories so tests exercise the real payload shape.
- For a Tier 2 (append-only) model factory, omit `updated_at`, set `created_at`/`occurred_at` via fake datetime.
