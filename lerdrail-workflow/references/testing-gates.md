# Test Gates — php / http / console / browser

## Table of Contents
- [The four gates](#the-four-gates)
- [PHP / HTTP / Console (all via `lerd test`)](#php--http--console-all-via-lerd-test)
- [Browser — Pest 4 native browser testing (NOT Dusk)](#browser--pest-4-native-browser-testing-not-dusk)
- [Test data: factories, seeders, fakes, mocks](#test-data-factories-seeders-fakes-mocks)
- [Test database](#test-database)
- [Gate matrix per module type](#gate-matrix-per-module-type)

How to *write* tests is in `laravel-conventions`'s [testing.md](../../laravel-conventions/references/testing.md). This reference defines the *gates* — what must pass, when, for this project.

## The four gates

| Gate | When required | Command | Notes |
|------|---------------|---------|-------|
| **PHP unit/feature** | every module | `lerd test` | covers Pest/PHPUnit; the default gate |
| **HTTP** | module exposes endpoints | `lerd test` | same suite — `$this->get/post`/`assertStatus`/`assertJson` run process-kernel |
| **Console** | module adds artisan commands | `lerd test` | same suite — `$this->artisan(...)` |
| **Browser** | module has UI interaction (as needed) | `lerd pest` (browser specs) | Pest 4 native browser testing |

Gates 1–3 run from one `lerd test` invocation — they're Pest test files, not separate commands. "Gate" here means: the relevant assertions *exist and pass*, not a separate run.

## PHP / HTTP / Console (all via `lerd test`)

```bash
lerd test                          # full suite = lerd artisan test
lerd test --filter=OrderTest       # one file
lerd test --testsuite=Feature      # one suite
lerd pest --coverage               # Pest directly + coverage
```

- **No `lerd exec`** — `lerd test` / `lerd artisan test` / `lerd pest` are the commands.
- These gates have **no Lerd-specific config** — they behave as standard Laravel because the test process runs inside the PHP-FPM container with the standard Laravel testing kernel.
- Write patterns in `laravel-conventions`'s [testing.md](../../laravel-conventions/references/testing.md).

## Browser — Pest 4 native browser testing (NOT Dusk)

This project uses **Pest 4's native browser testing** (`pestphp/pest-plugin-browser`, based on Playwright/musl Chromium), per Laravel's current recommendation for new projects. It is faster and simpler than Laravel Dusk, and Lerd has first-class support for it.

> Laravel's own guidance: *"Pest 4 now includes automated browser testing which offers significant performance and usability improvements compared to Laravel Dusk. For new projects, we recommend using Pest for browser testing."*

### One-time setup

```bash
lerd composer require --dev pestphp/pest-plugin-browser
lerd npm install playwright
lerd pest:browser install     # bakes musl Chromium into the PHP-FPM image + shims Playwright
lerd pest:browser doctor      # verify the setup
```

Notes:
- Only **Chromium** is supported (Alpine/musl has no Firefox/WebKit builds).
- Requires a current PHP version (frozen legacy tiers 7.4/8.0 are **not** supported).
- `lerd pest:browser remove` tears it down if needed.

Query the plugin via context7 `/pestphp/pest-plugin-browser`.

### Running

Browser specs are Pest tests using the browser plugin's API; run with the normal test command:

```bash
lerd pest --filter=browser        # or whatever your browser specs are named
```

### When to write browser tests

Browser tests are **expensive** (real browser, slower). Use them only for genuine UI interaction that HTTP tests can't cover:
- multi-step flows with client-side state (drag-drop, modals, dynamic forms)
- visual/behavioral assertions ("the button is disabled until the checkbox is checked")
- JS-dependent behavior

Don't write browser tests for pure API/JSON endpoints (HTTP gate) or command behavior (Console gate) — that's wasted effort.

## Test data: factories, seeders, fakes, mocks

- **`RefreshDatabase`** trait (per test or base TestCase) — re-migrates the `_testing` DB between tests.
- **Model factories** — entities via `Order::factory()->create([...])`. UUID v7 is auto-filled by `HasUuids`; never set `'id'`. See `laravel-conventions` models-and-casts.md.
- **`DatabaseSeeder`** — shared reference data (statuses, countries); seed in `setUp` when needed. `lerd artisan db:seed` for CI/manual seeding.
- **Fakes over mocks** (preferred): `Queue::fake()`, `Mail::fake()`, `Notification::fake()`, `Event::fake()`, `Http::fake([...])`, `Storage::fake(...)`. Assert dispatch/sent/queued without executing.
- **Mockery** — only when you can't fake the subsystem: `$this->mock(Service::class)->shouldReceive(...)`.

## Test database

`lerd db:create <name>` (and `lerd setup`) creates `<name>` **and** `<name>_testing` with identical credentials. Standard `phpunit.xml` sets `DB_DATABASE=testing` for the test env — no edit needed. The host inside the container is `lerd-postgres` (same as dev). See [lerd-dev host-networking](../../lerd-dev/references/host-networking.md).

## Gate matrix per module type

| Module type | PHP | HTTP | Console | Browser |
|-------------|-----|------|---------|---------|
| Pure service / value-object / cast | ✓ | — | — | — |
| API endpoint (JSON) | ✓ | ✓ | — | — |
| Console command | ✓ | — | ✓ | — |
| Job / listener / notification | ✓ | (assert dispatched/sent) | — | — |
| Inertia page (UI) | ✓ | ✓ (assertInertia) | — | ✓ (interaction) |
| Migration only | ✓ (rollback test) | — | — | — |

"Migration only" → also verify `lerd artisan migrate:rollback` works (reversibility gate, see quality-gates.md).
