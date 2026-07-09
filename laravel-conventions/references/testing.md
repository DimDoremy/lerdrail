# Testing — PHP / HTTP / Console

## Table of Contents
- [Pest, run via Lerd](#pest-run-via-lerd)
- [Test database](#test-database)
- [HTTP tests](#http-tests)
- [Console tests](#console-tests)
- [Unit tests](#unit-tests)
- [Test data: factories, seeders, RefreshDatabase](#test-data-factories-seeders-refreshdatabase)
- [Mocking](#mocking)

Browser testing (the 4th gate) is covered in `dev-workflow`'s testing-gates (Pest 4 native browser testing, not Dusk). Query Pest docs via context7 `/pestphp/docs`.

## Pest, run via Lerd

The project uses **Pest**. Run via Lerd so it uses the project's PHP container:

```bash
lerd test                          # = lerd artisan test
lerd test --filter=OrderTest
lerd test --testsuite=Feature
lerd pest                          # run Pest directly
lerd pest --coverage
lerd test --parallel               # parallel execution
```

There is no `lerd exec` — `lerd test` / `lerd artisan test` / `lerd pest` are the commands. The test *gates* (what must pass, when) are defined in `dev-workflow`'s [testing-gates.md](../../dev-workflow/references/testing-gates.md); this reference is about how to write them.

A Pest test:
```php
it('creates an order', function () {
    $user = User::factory()->create();

    $response = $this->actingAs($user)->post('/orders', [
        'number' => 'ORD-1',
        'amount_cents' => 5000,
        'currency' => 'USD',
    ]);

    $response->assertRedirect(route('orders.show', Order::first()));
    expect(Order::count())->toBe(1);
});
```

## Test database

`lerd db:create <name>` (and `lerd setup`) creates `<name>` **and** `<name>_testing` with identical credentials. The standard `phpunit.xml` sets `DB_DATABASE=testing` for the test env, so tests hit `<name>_testing` automatically. You do **not** edit `phpunit.xml`.

The host is `lerd-postgres` in the container (same as dev) — see [lerd-dev host-networking](../../lerd-dev/references/host-networking.md).

## HTTP tests

Process-kernel tests (no real HTTP request) — the bulk of feature tests:

```php
it('shows an order', function () {
    $order = Order::factory()->create();

    $this->actingAs($order->user)
        ->get(route('orders.show', $order))
        ->assertOk()
        ->assertInertia(fn (Assert $page) => $page
            ->component('Orders/Show')
            ->has('order', fn (Assert $o) => $o->where('number', $order->number)->etc())
        );
});

it('validates store', function () {
    $this->actingAs(User::factory()->create())
        ->post('/orders', ['amount_cents' => 'not-a-number'])
        ->assertSessionHasErrors(['amount_cents']);
});

it('paginates', function () {
    Order::factory()->count(30)->create();
    $this->get('/orders?page=2')->assertOk()->assertJsonStructure(['data','links']);
});

it('returns json for api', function () {
    $this->getJson('/api/orders')->assertOk()->assertJsonCount(0, 'data');
});
```

Useful assertions: `assertOk`, `assertStatus`, `assertRedirect`, `assertSessionHasErrors`, `assertJson`, `assertJsonStructure`, `assertJsonCount`, `assertInertia` (Inertia props assertion).

## Console tests

For Artisan commands — `expectsOutput`, exit codes, and DB side-effects:

```php
it('sends invoices for ready orders', function () {
    $ready = Order::factory()->create(['status' => OrderStatus::Paid]);
    $pending = Order::factory()->create(['status' => OrderStatus::Draft]);

    $this->artisan('invoices:send')
        ->expectsOutput('Sent 1 invoice(s)')
        ->assertSuccessful();

    expect($ready->fresh()->status)->toBe(OrderStatus::Invoiced);
    expect($pending->fresh()->status)->toBe(OrderStatus::Draft);
});

// when output is dynamic/unordered
it('prunes old logs', function () {
    $this->artisan('logs:prune')->assertSuccessful();
    expect(OrderEvent::where('occurred_at', '<', now()->subYear())->count())->toBe(0);
});
```

Use `Artisan::call(...)` / `Artisan::queue(...)` for imperative invocation in setup.

## Unit tests

For pure value-objects/casts/enums — no framework, fast:

```php
it('converts money to cents', function () {
    $m = new Money(12, 'USD');
    expect($m->amountInCents())->toBe(1200);
});
```

## Test data: factories, seeders, RefreshDatabase

- **`RefreshDatabase`** trait (or `DatabaseTransactions` for speed) resets state between tests by re-running migrations on the `_testing` DB. Put on a base `TestCase` or per-test.
- **Factories** for entities (see [models-and-casts.md](models-and-casts.md)); use `Model::factory()->create([...])` and `->make([...])` (persisted vs not).
- **`DatabaseSeeder`** for shared reference data (countries, statuses). Run conditionally or seed specific tables in `setUp`.
- **States** for variants: `User::factory()->admin()->create()`.

```php
uses(Illuminate\Foundation\Testing\RefreshDatabase::class);

beforeEach(function () {
    $this->user = User::factory()->create();
});
```

## Mocking

Prefer **faking** real subsystems over hand-mocking (more robust):

```php
// fake the queue — assert a job was dispatched without running it
Queue::fake();
$order = ...;
ProcessOrder::dispatch($order);
Queue::assertPushed(ProcessOrder::class, fn ($j) => $j->order->is($order));

// fake notifications, mail, events, http
Notification::fake(); Mail::fake(); Event::fake();
Http::fake(['stripe.com/*' => Http::response(['id' => 'ch_1'], 200)]);
Storage::fake('local');

// mock a collaborator only when you can't fake the subsystem
$svc = $this->mock(PaymentGateway::class);
$svc->shouldReceive('charge')->once()->with(5000, 'USD')->andReturn(['ok' => true]);
$this->app->instance(PaymentGateway::class, $svc);
```

Prefer `Queue::fake()` / `Http::fake()` over mocking the class under test — fakes exercise your real code paths with controlled boundaries, which catches more integration bugs.
