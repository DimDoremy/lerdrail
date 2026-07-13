---
name: lerdrail-workflow
description: Use when starting any feature or bugfix in this Laravel/Inertia/React/Lerd project, before writing code or committing. Orchestrates the superpowers workflow chain and injects project test gates, per-module acceptance, and branch discipline.
---

# Development Workflow

## The end-to-end flow

Every feature or bugfix follows this pipeline. This skill **orchestrates** the superpowers chain — it does not reimplement execution mechanics.

```
1. Request          (user describes a need)
2. brainstorming    → produces a spec/PRD (HARD-GATE: no implementation before approval)
                      REQUIRED SKILL: superpowers:brainstorming
3. writing-plans    → decomposes spec into a TDD plan.md (Task → steps → exact commands → expected output)
                      REQUIRED SKILL: superpowers:writing-plans
4. Branch           → create feature/<plan-name> off dev (NEVER main)   ← harness injects this
5. Execute          → subagent-driven-development: per-task subagent + dual review + commit embedded
                      REQUIRED SKILL: superpowers:subagent-driven-development
                      Each task internally runs: superpowers:test-driven-development (red/green)
                                                + superpowers:verification-before-completion (gate)
6. Module accept    → after the whole plan's tasks pass, run the four test gates + evidence → human confirms
                      ← harness injects this (per-module, between plans)
7. Finish branch    → superpowers:finishing-a-development-branch (merge feature → dev; dev→main is later/human)
                      REQUIRED SKILL: superpowers:finishing-a-development-branch
```

This skill fills the **three gaps** superpowers leaves open: (1) sequencing the chain, (2) project test gates, (3) branch discipline. Everything else — TDD micro-steps, subagent dispatch, dual review, verification, branch finishing — is delegated to superpowers and you MUST go through those skills rather than improvising.

## Gap 1 — Sequence the chain

superpowers skills don't auto-chain. When a feature request arrives, drive it through the stages in order, invoking each skill explicitly:

1. **Start with `superpowers:brainstorming`.** It produces a spec/PRD at `docs/superpowers/specs/YYYY-MM-DD-<feature>-design.md` and commits it. It has a HARD-GATE: **do not implement anything before the spec is approved.** Discuss the requirement with the user, settle scope, then approve.

2. **Then `superpowers:writing-plans`.** Turns the approved spec into `docs/superpowers/plans/YYYY-MM-DD-<feature>.md` — a structured plan: File Structure first, then Tasks, each Task broken into TDD micro-steps (write failing test → run & see fail → write minimal impl → run & see pass → commit), with **exact commands and expected output** (no placeholders).

3. **Branch (see Gap 3) before executing.**

4. **Then `superpowers:subagent-driven-development`** to execute the plan (preferred over inline `executing-plans` because per-task subagents with dual review yield higher quality). It dispatches a fresh subagent per task, runs spec-compliance review then code-quality review, fixes-and-reviews until both pass, marks the task done, continues. Commits are embedded as plan steps.

5. **After the plan completes** → module acceptance (Gap 2) → `superpowers:finishing-a-development-branch`.

## Gap 2 — Project test gates

`superpowers:verification-before-completion` enforces "no completion claims without fresh evidence." **This** layer defines *what counts as evidence* for this project — the four test gates. Detail in [references/testing-gates.md](references/testing-gates.md).

| Gate | When | Command |
|------|------|---------|
| **PHP (unit/feature)** | every module | `lerd test` (= `lerd artisan test`) |
| **HTTP** | module has endpoints | `lerd test` (same suite; process-kernel assertions) |
| **Console** | module has artisan commands | `lerd test` (same suite; `$this->artisan`) |
| **Browser** | module has UI interaction (as needed) | Pest 4 native browser testing — **not Dusk** |

**Browser testing uses Pest 4 native browser testing** (`pestphp/pest-plugin-browser`, Playwright-based), per Laravel's current recommendation for new projects — not Laravel Dusk/Selenium. One-time setup: `lerd composer require --dev pestphp/pest-plugin-browser` + `lerd npm install playwright` + `lerd pest:browser install` (bakes musl Chromium into the FPM image) + `lerd pest:browser doctor`. See [testing-gates.md](references/testing-gates.md).

Test data: `RefreshDatabase` trait + model factories (UUID v7 auto-filled), `DatabaseSeeder` for shared fixtures, `Queue::fake()`/`Http::fake()`/Mockery for collaborators. The `_testing` database is auto-created by `lerd db:create`.

**Per-module acceptance** (the chosen granularity): a plan (a functional module) is accepted only when all its tasks pass + the relevant test gates are green + verification-before-completion evidence is in-hand. Then a human confirms before the next plan starts. Detail in [references/module-acceptance.md](references/module-acceptance.md).

## Gap 3 — Branch discipline

**All development happens on a `feature/<plan-name>` branch off `dev`. Never commit to `main`. Never commit to `dev` directly either — feature branches merge into `dev`.**

- `main` — production-ready; merged into only after full acceptance, by a human.
- `dev` — integration branch; the base for feature branches.
- `feature/<plan-name>` — one branch per plan/feature; created from `dev`; merged back to `dev` when the plan is accepted.

```
git checkout dev && git pull
git checkout -b feature/<plan-name>     # one per plan, BEFORE executing
# ... work, commit per task ...
# on acceptance:
git checkout dev && git merge --no-ff feature/<plan-name>
```

This is **stricter** than superpowers' default ("don't touch main without consent") — here `dev` is the integration line and `main` is fully hands-off during development. superpowers' `finishing-a-development-branch` handles the merge; this skill ensures the branch lineage is `feature → dev`, not `feature → main`.

If using git worktrees (`superpowers:using-git-worktrees`), create the worktree from `dev` and the feature branch within it.

## Debugging maps onto superpowers

When a task fails or a bug appears, `superpowers:systematic-debugging` is the method (root cause, not symptom-patching). Its *instruments* here are the Lerd + Boost tools:

- **App-level root cause** → Boost (Error Tracking, Database Queries, Configuration, schema, tinker).
- **Runtime debugging** → Lerd (dump/dd → query viewer → profiler → worker-heal → logs). See [lerd-dev/debugging.md](../lerd-dev/references/debugging.md).

Pair them: Boost tells you *what* the app is doing wrong; Lerd instruments show *where* in the request/worker it happens.

## Pre-commit quality gates

Before any commit that the plan's steps trigger, and before module acceptance, run the checklist in [references/quality-gates.md](references/quality-gates.md): `lerd test`, `lerd npm run build`, `lerd artisan pint --test` (or phpstan), dump cleanup (`lerd dump off` + no stray `dump()`), worker health, `lerd artisan migrate:status`. Migrations must be reversible (`down()` reverses `up()`).

## Quick reference

| Stage | Skill / action |
|-------|----------------|
| Understand & scope the request | `superpowers:brainstorming` → spec/PRD |
| Decompose into TDD tasks | `superpowers:writing-plans` → plan.md |
| Create branch | `feature/<plan-name>` off `dev` |
| Execute per-task | `superpowers:subagent-driven-development` |
| TDD inside a task | `superpowers:test-driven-development` |
| Verify before claiming done | `superpowers:verification-before-completion` + test gates |
| Module accepted | four gates green + human confirm → next plan |
| Finish | `superpowers:finishing-a-development-branch` (merge → dev) |
| Debug | `superpowers:systematic-debugging` + Boost/Lerd instruments |

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Implementing before the spec is approved | `brainstorming` has a HARD-GATE — respect it |
| Committing to `main` or `dev` directly | Use `feature/<plan-name>` off `dev` |
| Skipping browser tests for a UI module | Run the browser gate when the module has interaction |
| Using Dusk/Selenium | This project uses Pest 4 native browser testing |
| Claiming "done" without test evidence | `verification-before-completion` — show fresh output |
| Letting one plan bleed into the next | Module acceptance is a hard stop with human confirmation |
| Running `php artisan test` from host | Use `lerd test` |
| Leaving `dump()` in / `lerd dump on` | Clean before commit |
