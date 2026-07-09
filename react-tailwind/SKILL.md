---
name: react-tailwind
description: Use when building the React + Tailwind frontend of an Inertia.js app—page components under resources/js/Pages, shared layouts, component structure, Tailwind utility conventions, Vite build/SSR setup, and TypeScript types for JsonB-backed API payloads. Covers when to extract components, design token usage, and SSR via Lerd's FrankenPHP worker runtime. Laravel-side docs via Boost; React/Tailwind docs via context7.
---

# React + Tailwind (Inertia frontend)

## How this skill relates to other layers

- **Laravel-side of Inertia** (controller, `Inertia::render`, shared data, `useForm` server contract) → `laravel-conventions`'s [inertia-react.md](../laravel-conventions/references/inertia-react.md).
- **React/Tailwind API specifics** → context7 (`/reactjs/react.dev`, `/tailwindlabs/tailwindcss.com`).
- **Building/SSR under Lerd** → `lerd npm run build`, `lerd runtime frankenphp --worker`.
- **TypeScript types for API payloads** → derived from the JsonB-backed Resources on the backend; see [references/types-for-jsonb.md](references/types-for-jsonb.md).

## Project layout

The frontend lives inside the Laravel repo under `resources/js/`:

```
resources/js/
├── Pages/            # Inertia page components — one per route (the "screens")
│   ├── Orders/
│   │   ├── Index.tsx
│   │   └── Show.tsx
│   └── Dashboard.tsx
├── Layouts/          # shared page shells (app chrome, nav, sidebar)
├── Components/       # reusable presentational + interactive components
│   ├── ui/           # design-system primitives (Button, Input, Card, Modal)
│   └── orders/       # feature-scoped components
├── Hooks/            # custom hooks
├── lib/              # pure helpers (formatting, api client wrappers)
├── types/            # shared TS types (mirrors API Resources)
└── ssr.tsx           # SSR entry (when SSR enabled)
```

Page composition, layout usage, and the "when to extract a component" decision in [references/component-structure.md](references/component-structure.md).

## Data flow (Inertia)

Pages receive props from `Inertia::render` via `usePage()`:

```tsx
import { Head, Link } from '@inertiajs/react';
import { PageProps } from '@/types';

export default function OrdersShow({ order }: PageProps<{ order: OrderResource }>) {
  return (
    <>
      <Head title={`Order ${order.number}`} />
      <h1>{order.number}</h1>
      <Link href="/orders">Back</Link>
    </>
  );
}
```

Forms use `useForm` (see `laravel-conventions` inertia-react.md). Don't introduce a client state library (Redux/Zustand) unless cross-page client state is genuinely needed — Inertia props + `useForm` cover most apps.

## Tailwind conventions

Utility-first, but disciplined:

- **Design tokens in the theme** — colors, spacing, font sizes live in `tailwind.config` / CSS variables, not as one-off magic numbers in classes.
- **Compose, don't pile** — when a class string repeats across 2+ places, extract a component (or, rarely, an `@apply` rule), not a wall of utilities copied everywhere.
- **Responsive order** — mobile-first: `block md:flex lg:grid`, not `lg:grid md:flex block`.
- See [references/tailwind-conventions.md](references/tailwind-conventions.md) for token usage, dark mode, and the extract-component threshold.

## Build & SSR (Lerd / Vite)

```bash
lerd npm install
lerd npm run dev      # Vite dev server with HMR
lerd npm run build    # production build (goes to public/build)
```

SSR (optional): `lerd runtime frankenphp --worker` serves server-rendered React via Laravel; entry at `resources/js/ssr.tsx`. Setup details in [references/vite-ssr.md](references/vite-ssr.md).

## TypeScript types for JsonB payloads

The backend exposes JsonB through API Resources (see `postgres-valkey` data architecture). Mirror those shapes as TS types so the frontend is typed end-to-end. Patterns (plain interface vs zod) in [references/types-for-jsonb.md](references/types-for-jsonb.md).

## Quick reference

| Want to… | Do |
|----------|----|
| Add a page | `resources/js/Pages/<Feature>/<Name>.tsx`, render via `Inertia::render` |
| Share page chrome | `Layouts/AppLayout.tsx`, wrap pages with it |
| Add a reusable bit of UI | `Components/ui/<Name>.tsx` (primitive) or `Components/<feature>/` |
| Submit a form | `useForm` from `@inertiajs/react` |
| Look up a hook/util | context7 `/reactjs/react.dev` |
| Look up a Tailwind class | context7 `/tailwindlabs/tailwindcss.com` |
| Build for production | `lerd npm run build` |
| Enable SSR | `lerd runtime frankenphp --worker` + ssr.tsx entry (see vite-ssr.md) |

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Duplicating a 10-class string across files | Extract a component (or `@apply` in a component CSS module) |
| Magic color/spacing literals in classes | Use theme tokens |
| `fetch` + `useState` for a form | `useForm` |
| Adding Redux for one piece of form state | Inertia props + `useForm` is enough |
| Untyped Inertia props (`any`) | Type via `PageProps<T>` mirroring the Resource |
| Mobile-last responsive classes | Write mobile-first (`sm:` / `md:` upward) |
| Running `npm run build` on host | Use `lerd npm run build` (pinned Node) |
