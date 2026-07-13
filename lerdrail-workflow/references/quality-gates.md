# Pre-commit Quality Gates

## Table of Contents
- [The checklist](#the-checklist)
- [Migration safety](#migration-safety)
- [Cleanup of debug instruments](#cleanup-of-debug-instruments)
- [Worker health](#worker-health)
- [Running the gates](#running-the-gates)

Run this checklist before every commit the plan triggers (the plan's commit steps assume these pass) and again at module acceptance. It complements the test gates in [testing-gates.md](testing-gates.md).

## The checklist

| # | Gate | Command | Pass when |
|---|------|---------|-----------|
| 1 | Tests | `lerd test` | 0 failures, exit 0 (covers php/http/console) |
| 2 | Browser tests (if UI) | `lerd pest --filter=browser` | 0 failures |
| 3 | Frontend build | `lerd npm run build` | exit 0, build emitted to `public/build` |
| 4 | Code style | `lerd artisan pint --test` | clean (or `lerd pint` to fix) |
| 5 | Static analysis (if configured) | `lerd phpstan analyse` | 0 errors |
| 6 | Dump cleanup | `lerd dump off`; `grep -rn "dump(\|dd(" app/` | bridge off; no stray `dump()`/`dd()` calls |
| 7 | Worker health | `lerd worker list` (or MCP `worker` `health`) | no `failed` workers (heal if any) |
| 8 | Migration state | `lerd artisan migrate:status` | all pending run; rollback tested for new ones |
| 9 | Reversibility | `lerd artisan migrate:rollback` for new migrations | rolls back cleanly |

## Migration safety

Every migration must be **reversible**:

- `up()` and `down()` both present; `down()` truly reverses `up()` in reverse order (drop indexes before columns/types).
- For DDL added via `DB::statement` (CREATE TYPE, generated columns, custom indexes, triggers), `down()` drops them — see `postgres-valkey` migration-templates.md.
- Snapshot before risky changes: `lerd db:snapshot pre-<feature>`; restore to test rollback: `lerd db:restore pre-<feature>`.
- Verify with `lerd artisan migrate:status` (all run) and a `migrate:rollback` dry test on the new migrations.
- Data migrations (moving data between columns/tables) go in a **separate** migration from schema changes where possible, and must be idempotent and reversible (or at least document why not).

## Cleanup of debug instruments

Debugging leaves traces that must not ship:
- `lerd dump off` — turn the dump bridge off (it's a global flag; leaving it on is harmless to the app but signals unfinished work).
- `grep -rn "dump(\|dd(" app/` — remove any `dump()`/`dd()` added during debugging (leave intentional ones only with a comment, rare).
- `lerd profile off` — disable the SPX profiler.
- `lerd xdebug off` — disable xdebug if enabled for the task.
- Remove `Log::debug` / `ray()` / similar breadcrumbs.

## Worker health

Queue/schedule/horizon workers can enter `failed` state (restart-rate-limit cascade or app crash loop). Before accepting a module:
```bash
lerd worker list                 # or MCP worker health
lerd worker heal                 # reset+start any failed (one attempt each)
```
If a worker re-fails, the root cause is in its logs (`lerd worker logs <name>` / MCP `logs` `worker:<name>`) — fix before accepting, don't heal-and-pray. See [lerd-dev/debugging.md](../../lerd-dev/references/debugging.md).

## Running the gates

A one-shot for the common case:
```bash
lerd test && \
lerd npm run build && \
lerd artisan pint --test && \
lerd dump off && \
lerd artisan migrate:status
```
Add `lerd phpstan analyse` if configured, `lerd pest --filter=browser` if the module has UI, and check `lerd worker list`. Snapshot first (`lerd db:snapshot pre-accept`) so you can test rollback without losing data.

If any gate fails: do **not** commit / claim done. Fix the cause (`systematic-debugging`), re-run the failed gate, then proceed. `verification-before-completion`: claims require fresh evidence — re-run, don't cite stale results.
