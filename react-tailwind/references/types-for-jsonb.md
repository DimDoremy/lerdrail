# TypeScript Types for JsonB Payloads

## Table of Contents
- [Why this matters](#why-this-matters)
- [Mirror Resources, not models](#mirror-resources-not-models)
- [Plain interfaces (simple)](#plain-interfaces-simple)
- [zod schemas (validated runtime)](#zod-schemas-validated-runtime)
- [JsonB as a typed sub-object](#jsonb-as-a-typed-sub-object)
- [Optional / evolving fields](#optional--evolving-fields)
- [The PageProps pattern](#the-pageprops-pattern)

## Why this matters

The backend exposes data through API Resources (see `laravel-conventions`), and several of those carry **JsonB** columns (`meta`, `payload`, `context`, …) — semi-structured data per the [data-architecture principle](../../postgres-valkey/references/data-architecture.md). If the frontend treats those as `any`, the JsonB flexibility becomes a source of runtime crashes. Type them end-to-end.

## Mirror Resources, not models

Type the **API Resource** shape (what the frontend actually receives), not the Eloquent model (which has DB-internal fields the client never sees). One TS type per Resource:

```ts
// types/resources.ts
import type { OrderStatus } from './enums';

export interface OrderResource {
  id: string;                // UUID v7
  number: string;
  status: OrderStatus;
  amount_cents: number;
  currency: string;
  meta: OrderMeta;           // the JsonB column, typed below
  created_at: string;        // ISO timestamp serialized as string
  user?: UserResource;
}
```

Keep these in `resources/js/types/` (or `types/resources.ts`) and import from pages/components.

## Plain interfaces (simple)

For a JsonB column with a known, stable shape, a plain interface is enough:

```ts
// types/resources.ts
export interface OrderMeta {
  channel?: 'web' | 'api' | 'mobile';
  notes?: string;
  tags?: string[];
}

export interface OrderResource {
  // ...
  meta: OrderMeta;
}
```

Access safely in components:
```tsx
const channel = order.meta.channel ?? 'unknown';
```

This covers most JsonB usage — the "restless payload" is still typed where its shape is known, and optional (`?`) where it evolves.

## zod schemas (validated runtime)

When the JsonB payload comes from an **untrusted or rapidly-evolving** source (a webhook body, user-imported data, a third-party integration) and you want runtime validation (not just compile-time types), use **zod**. Define the schema once; derive the type from it:

```ts
// types/schemas.ts
import { z } from 'zod';

export const OrderMetaSchema = z.object({
  channel: z.enum(['web', 'api', 'mobile']).optional(),
  notes: z.string().optional(),
  tags: z.array(z.string()).default([]),
  // tolerate unknown extra keys from the evolving payload
}).passthrough();

export type OrderMeta = z.infer<typeof OrderMetaSchema>;

// validate at the trust boundary (e.g. when receiving a webhook import)
const parsed = OrderMetaSchema.safeParse(raw);
if (!parsed.success) { /* handle */ }
const meta: OrderMeta = parsed.data;
```

Rule of thumb: **plain interface for stable JsonB, zod for evolving/untrusted JsonB.** Don't zod everything — it adds weight where a type is sufficient.

## JsonB as a typed sub-object

If a JsonB column holds a structured value-object (e.g. an `Address`), mirror it as its own interface — it's a concept, treat it as one:

```ts
export interface Address {
  line1: string;
  line2?: string;
  city: string;
  region?: string;
  postal_code: string;
  country: string;          // ISO-3166 alpha-2
}

export interface OrderResource {
  // ...
  shipping_address: Address | null;   // JsonB column, nullable
  billing_address: Address | null;
}
```

This matches a custom value-object cast on the backend (see `laravel-conventions` models-and-casts.md) — the cast serializes the object to JSON, the TS interface mirrors it.

## Optional / evolving fields

JsonB payloads evolve by design. Encode that in the types:
- Use `?` for fields that may be absent on older records.
- Use union/`null` (`Address | null`) for nullable.
- For a grab-bag "extension" object, a typed index: `[key: string]: unknown;` — better than `any`.
- When promoting a JsonB key to a real column (the →normalized-SQL migration path), the Resource and TS type change in lockstep: the field moves out of `meta` into a top-level typed field.

## The PageProps pattern

Inertia injects the page props; type them so every page is type-checked against the backend contract:

```ts
// types/index.ts
import type { OrderResource } from './resources';

export interface PageProps<T = Record<string, unknown>> {
  auth: { user: UserResource | null };
  flash?: { success?: string; error?: string };
  // ...shared props from HandleInertiaRequests::share
} & T;

// usage in a page
export default function OrdersShow({ order }: PageProps<{ order: OrderResource }>) {
  // order is fully typed, including its JsonB meta
}
```

This makes the Inertia controller→props→page contract compile-time-checked: if a Resource changes, TS flags every page that needs updating.

## Common mistakes

| Mistake | Fix |
|---------|-----|
| `any` for JsonB | Type it (interface or zod) |
| Typing the Eloquent model instead of the Resource | Type what the client receives |
| zod everywhere | Plain interface for stable shapes; zod for evolving/untrusted |
| Forgetting optional `?` on evolving fields | JsonB evolves — mark fields optional |
| Untyped `usePage().props` | Use `PageProps<T>` |
