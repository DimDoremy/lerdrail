# lerdrail

> **Rails to keep vibe-coding Laravel on track.**
> A 5-skill harness pack for building Laravel apps on [Lerd](https://lerd.sh) — Laravel + Inertia/React/Tailwind on PostgreSQL (with a path to ClickHouse), driven by Boost + Lerd MCP, context7, and the superpowers workflow skills.

`lerdrail` is a project-local **skills pack** (not a library, not a package you `composer require`). You drop it into a Laravel project's `.zcode/skills/`, commit it to git, and your AI coding agent (ZCode / Claude Code / any agent that loads Agent Skills) gains a coherent set of conventions, decisions, and workflow discipline tuned to this exact stack — instead of rediscovering the right way to do things on every turn.

## Why

Vibe-coding on a real stack fails in predictable ways: the agent puts `127.0.0.1` in `.env` and the container can't reach the DB; it models everything as flat columns and locks you out of analytics later; it claims "done" without running tests; it commits to `main`. `lerdrail` encodes the corrections up front so the agent doesn't make them.

It coordinates five distinct AI capability layers so each does what it's best at:

```
Boost MCP    → Laravel app introspection (schema/routes/config/logs/tinker/doc-search)
Lerd MCP/CLI → Environment & runtime (services/sites/PHP version/workers/dump/profiler/tests)
context7     → Non-Laravel docs (React/Tailwind/Inertia/PostgreSQL/Valkey/ClickHouse/Pest)
lerdrail     → Conventions + data architecture + workflow discipline (this pack)
superpowers  → Execution engine (brainstorm → plan → subagent execute → verify → finish)
```

## What's inside

Five focused skills, each with a precise trigger and progressive-disclosure references:

| Skill | Owns |
|-------|------|
| **`lerd-dev`** | The five-way division of labor, the 11 grouped Lerd MCP tools, the container-vs-host networking trap, site lifecycle, debugging loop (dump → query → profiler → worker-heal) |
| **`laravel-conventions`** | Artisan codegen, Eloquent `HasUuids` + JsonB casts, form requests / API resources, Inertia render/props flow, Pest tests (php/http/console) |
| **`postgres-valkey`** | The data-architecture first principle (see below), PG features (JSONB / arrays / enums / GIN / FTS / CTEs), migration templates, ClickHouse migration path, Redis/Valkey + Horizon |
| **`react-tailwind`** | Inertia page/layout/component structure, Tailwind token discipline, Vite build & SSR via FrankenPHP, TypeScript types mirroring JsonB-backed resources |
| **`dev-workflow`** | Orchestrates the superpowers chain with three project layers: four test gates, per-module acceptance, and `feature/<plan>` off `dev` branch discipline (never `main`) |

### The data-architecture first principle

Every table follows a **wide-table + JsonB middle form** designed to migrate **bidirectionally** — toward normalized SQL (promote JsonB keys to columns) or toward ClickHouse (keep append-only wide tables) — via two hard rules:

- **Tiered modeling** — core business entities are normalized (FKs, JOINs, soft-deletes); event/log/behavioral streams are append-only wide tables (no UPDATE/DELETE, time-indexed, ClickHouse-first).
- **Unified UUID v7 primary keys** on all new tables — time-ordered (good for PG B-tree *and* ClickHouse `ORDER BY`), so a phased migration never hits a key-type clash.

### The development workflow

```
request → brainstorming (spec/PRD, HARD-GATE)
        → writing-plans (TDD plan.md)
        → feature/<plan> branch off dev
        → subagent-driven-development (per-task TDD + dual review + commit)
        → module acceptance (4 test gates + evidence + human confirm)
        → finishing-a-development-branch (merge → dev; dev → main is human/later)
```

Browser testing uses **Pest 4 native browser testing** (Playwright-based), per Laravel's current recommendation for new projects — not Dusk/Selenium.

## Requirements

- A local **Lerd** install (rootless Podman, `*.test` domains) — https://lerd.sh
- A Laravel project with **Laravel Boost** (`composer require laravel/boost --dev && php artisan boost:install`)
- The **superpowers** plugin installed in your agent (provides `brainstorming`, `writing-plans`, `subagent-driven-development`, etc.)
- The **context7** MCP server configured
- An agent that loads Agent Skills (ZCode, Claude Code, …)

## Install

`lerdrail` is meant to live **inside** a project and travel with its git history — clone-and-go for teammates, same philosophy as `lerd mcp:inject`.

```bash
# from your Laravel project root
mkdir -p .zcode/skills
cd .zcode/skills
git clone https://github.com/DimDoremy/lerdrail.git
# or, to vendor it into the repo:
# cp -r lerdrail/* . && rm -rf lerdrail
```

Each skill is a directory with a `SKILL.md` (name + description trigger) and a `references/` folder loaded on demand. Nothing to build, nothing to configure beyond the requirements above.

## Project layout

```
lerdrail/
├── lerd-dev/                 # core orchestration skill
│   ├── SKILL.md
│   └── references/           # mcp-division, host-networking, mcp-tools, workflow, debugging, context7-queries
├── laravel-conventions/
│   ├── SKILL.md
│   └── references/           # models-and-casts, inertia-react, artisan-codegen, testing
├── react-tailwind/
│   ├── SKILL.md
│   └── references/           # component-structure, tailwind-conventions, vite-ssr, types-for-jsonb
├── postgres-valkey/
│   ├── SKILL.md
│   └── references/           # data-architecture (core), postgres-features, migration-templates, clickhouse-migration, cache-queue
└── dev-workflow/
    ├── SKILL.md
    └── references/           # testing-gates, module-acceptance, quality-gates
```

## Status

Early. The pack encodes one team's conventions; adapt the data-architecture rules, branch policy, and test gates to your own project. The skill files are plain Markdown — edit them directly.

## License

MIT
