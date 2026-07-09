# Module Acceptance & Branch Discipline

## Table of Contents
- [What a "module" is](#what-a-module-is)
- [The acceptance flow](#the-acceptance-flow)
- [Boundaries with subagent-driven-development](#boundaries-with-subagent-driven-development)
- [Branch lineage](#branch-lineage)
- [When to stop and ask](#when-to-stop-and-ask)

## What a "module" is

A **module** = one `superpowers:writing-plans` plan. It's the unit of acceptance: a coherent feature or bugfix decomposed into Tasks in a single `docs/superpowers/plans/YYYY-MM-DD-<feature>.md`. The plan is accepted (or not) as a whole; you don't move to the next plan until the current one is accepted.

## The acceptance flow

```
Plan execution finishes (all Tasks done via subagent-driven-development)
   │
   ▼
1. Run test gates relevant to this module          (testing-gates.md matrix)
     lerd test                                   # php/http/console
     lerd pest --filter=browser                  # browser, if UI interaction
   │
   ▼
2. Run pre-commit quality gates                   (quality-gates.md)
     lerd npm run build
     lerd artisan pint --test   (or phpstan)
     lerd dump off + grep for stray dump()/dd()
     lerd worker list (heal if failed)
     lerd artisan migrate:status
   │
   ▼
3. Apply verification-before-completion            (superpowers)
     IDENTIFY the command → RUN it → READ output → VERIFY it proves the claim
     No "done" without fresh evidence in this turn.
   │
   ▼
4. Human confirms acceptance
     Present: test output, gate results, what was built (vs spec), risks.
     Wait for explicit confirmation.
   │
   ▼
5. On acceptance: finish the branch                (superpowers:finishing-a-development-branch)
     merge feature/<plan-name> → dev
   │
   ▼
6. Only then start the next plan (new feature/<plan-name> off updated dev)
```

Steps 1–3 produce **evidence**; step 4 is the **human gate**; step 5 is the **merge**. Do not start the next plan (step 6) before acceptance — that's the whole point of per-module acceptance.

## Boundaries with subagent-driven-development

`superpowers:subagent-driven-development` runs **continuously** through one plan — it dispatches a subagent per Task, dual-reviews, fixes, and continues without stopping between Tasks. That's correct *within* a plan.

The **acceptance gate sits between plans**, not between Tasks. So:
- Inside a plan: let subagent-driven-development run (don't interrupt between tasks).
- Between plans: hard stop, run gates, get human confirmation, merge, then next plan.

If the user wants Task-level checkpoints too, that's a deliberate override of the continuous-execution principle — confirm explicitly before inserting stops.

## Branch lineage

```
main ──────────────────────────────────────────────── (hands-off during dev)
        │
        dev ──────┬───────────────┬──────────────    (integration; feature branches merge here)
                  │               │
        feature/plan-a     feature/plan-b           (one per plan, off dev)
```

Rules:
- **`main` is never committed to during development.** It receives merges only after full acceptance + human decision, later.
- **`dev` is the integration branch.** Feature branches are created from it and merge back into it.
- **`feature/<plan-name>` is the working branch**, one per plan, created **before** execution begins.

Creation, before executing a plan:
```bash
git checkout dev && git pull --ff-only
git checkout -b feature/<plan-name>
```

Merge on acceptance (via `superpowers:finishing-a-development-branch`):
```bash
git checkout dev
git merge --no-ff feature/<plan-name>
git branch -d feature/<plan-name>     # (or keep, per the finishing skill's options)
```

With git worktrees (`superpowers:using-git-worktrees`): create the worktree from `dev`, the feature branch lives in the worktree; finish merges back to `dev` in the main checkout.

## When to stop and ask

Stop and get human input (don't guess) when:
- The spec is ambiguous or the request changed mid-plan → back to `brainstorming`.
- A Task is BLOCKED (subagent returns BLOCKED) → per `subagent-driven-development`, never ignore or auto-retry-empty.
- A test/gate fails repeatedly and the root cause is unclear → `systematic-debugging`; if still stuck, ask.
- The module is done but acceptance evidence is missing or contradictory → re-verify before claiming.
- About to merge to `main` → that's always a human decision, never automatic.
