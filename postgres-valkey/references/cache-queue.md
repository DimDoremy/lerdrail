# Redis / Valkey — Cache, Queue, Horizon

## Table of Contents
- [Valkey vs Redis](#valkey-vs-redis)
- [Lerd host convention](#lerd-host-convention)
- [Cache](#cache)
- [Queue](#queue)
- [Horizon](#horizon)
- [Locks & rate limiting](#locks--rate-limiting)
- [Using Valkey instead of default Redis](#using-valkey-instead-of-default-redis)

Query the current Valkey docs via context7 `/websites/valkey_io`; Redis commands apply identically for Laravel usage.

## Valkey vs Redis

Valkey is a Redis fork (after Redis's license change) and is **wire-compatible** with the Redis protocol. From Laravel's perspective there is no difference: same config keys, same commands, same `phpredis`/`predis` drivers. Lerd ships **Redis 7-alpine** by default; the project uses the generic "Redis/Valkey" framing so either can drop in.

## Lerd host convention

Inside the PHP-FPM container (where Laravel runs), the host is **`lerd-redis`**, NOT `127.0.0.1`. `lerd env` writes it:

```env
REDIS_HOST=lerd-redis
REDIS_PORT=6379
QUEUE_CONNECTION=redis
CACHE_STORE=redis
```

Host-side tools (a Redis GUI, `redis-cli` from your terminal) use `127.0.0.1:6379`. See [lerd-dev host-networking](../../lerd-dev/references/host-networking.md).

Start/check:
```bash
lerd service start redis
lerd service status redis
```

When `QUEUE_CONNECTION=redis`, Lerd verifies `lerd-redis` is running before starting queue workers, and prompts you to start it if not.

## Cache

```php
// config/cache.php — 'default' => env('CACHE_STORE', 'database')
// .env: CACHE_STORE=redis
$value = Cache::remember('users.summary', 300, fn () => User::count());
Cache::tags(['users','reports'])->put('key', $data, 600);   // tag-based invalidation
Cache::tags(['users'])->flush();
```

Redis supports **cache tags** (database/file don't) — use them for grouped invalidation (all "user:123" cache entries under a `users` tag).

Atomic operations:
```php
Cache::add('once', $value, 60);          // set-if-not-exists
Cache::lock('process:order:'.$id, 60)->block(5, function () { /* exclusive */ });
```

## Queue

```env
QUEUE_CONNECTION=redis
```
```php
// dispatch
ProcessReport::dispatch($report)->onQueue('reports');
// chained
Bus::chain([ OptimizeImage::dispatch($img), GenerateThumbnail::dispatch($img) ])->dispatch();
// delayed
SendReminder::dispatch($invoice)->delay(now()->addHour());
```

Run workers under Lerd (systemd-user units that auto-restart and survive reboots):
```bash
lerd queue:start
lerd queue:stop
lerd worker list
```

Code-reload: when `.env` / `composer.*` / `.php-version` change, Lerd signals `php artisan queue:restart` inside the FPM container (2s debounce) so workers pick up new code.

## Horizon

For visibility into Redis queues (dashboard, metrics, auto-scaling):

```bash
lerd horizon:start
lerd horizon:stop
lerd horizon:reload
```
Configure per the Horizon package; its dashboard mounts at `/horizon` (secure it in production). Horizon supervises the queue workers itself — when using Horizon, start Horizon instead of plain `queue:start`.

## Locks & rate limiting

```php
// atomic lock
Cache::lock('export:'.$user->id, 120)->block(5, function () {
    ExportReport::run($user);
});
// rate limit
RateLimiter::for('uploads', fn (Request $r) => Limit::perMinute(30)->by($r->user()->id));
// throttle a job
ThrottlesExceptions::class middleware on the job, or ReleaseAfter::class for backoff.
```

Redis is the natural backend for distributed locks and rate limiting (atomic, shared across workers).

## Using Valkey instead of default Redis

If you specifically want the Valkey image instead of Lerd's default Redis 7:

1. Check for a preset: `lerd service search valkey`.
2. If none, add a custom service. `lerd service add` accepts an OCI image with env injection — point it at the Valkey image (e.g. `valkey/valkey:7`), expose port 6379, and name the service so the container host is `lerd-valkey` (then set `REDIS_HOST=lerd-valkey` in `.env`). See `lerd service add --help` and the custom-services docs at `https://lerd.sh/usage/custom-services`.

In practice, because Valkey is wire-compatible, the project's default Redis 7 is fine and "Redis/Valkey" in code/config is a no-op distinction until you deliberately swap the image. The generic framing in this skill means you never have to rewrite cache/queue code if you switch.
