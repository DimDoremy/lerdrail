# Component Structure

## Table of Contents
- [Pages, Layouts, Components](#pages-layouts-components)
- [Page anatomy](#page-anatomy)
- [Layouts](#layouts)
- [Component tiers](#component-tiers)
- [When to extract a component](#when-to-extract-a-component)
- [Props and typing](#props-and-typing)
- [Custom hooks](#custom-hooks)

## Pages, Layouts, Components

Three layers, distinct jobs:

- **Pages** (`Pages/`) — one component per route, the "screens". Receive Inertia props, compose Layouts + Components. Thin: orchestration, not logic.
- **Layouts** (`Layouts/`) — persistent page chrome (app shell, nav, sidebar, footer). Pages opt into a layout; the layout renders `children`.
- **Components** (`Components/`) — reusable UI. Split into `ui/` (design-system primitives, feature-agnostic) and feature folders (`Components/orders/`).

```
Pages/Orders/Show.tsx          # screen: receives { order }, renders layout + components
Layouts/AppLayout.tsx          # chrome: nav + sidebar + <main>{children}</main>
Components/ui/Button.tsx       # primitive: <Button variant="primary">
Components/ui/DataTable.tsx    # primitive: generic table
Components/orders/StatusBadge.tsx  # feature: knows OrderStatus
```

## Page anatomy

```tsx
import { Head } from '@inertiajs/react';
import AppLayout from '@/Layouts/AppLayout';
import { StatusBadge } from '@/Components/orders/StatusBadge';
import type { PageProps } from '@/types';
import type { OrderResource } from '@/types/resources';

interface Props extends PageProps {
  order: OrderResource;
}

export default function OrdersShow({ order }: Props) {
  return (
    <AppLayout>
      <Head title={`Order ${order.number}`} />
      <header className="flex items-center justify-between">
        <h1 className="text-2xl font-semibold">{order.number}</h1>
        <StatusBadge status={order.status} />
      </header>
      {/* ... */}
    </AppLayout>
  );
}

OrdersShow.layout = (page: React.ReactNode) => <AppLayout children={page} />;
// (the per-page `layout` static is optional — many set the layout in the component body as above)
```

Pages stay focused on *what to show*; they delegate *how it looks* to Components and *where it sits* to Layouts.

## Layouts

```tsx
// Layouts/AppLayout.tsx
import { Link } from '@inertiajs/react';
import { usePage } from '@inertiajs/react';

export default function AppLayout({ children }: { children: React.ReactNode }) {
  const { auth } = usePage().props;
  return (
    <div className="min-h-screen flex">
      <aside className="w-64 border-r p-4">{/* nav */}</aside>
      <main className="flex-1 p-6">{children}</main>
    </div>
  );
}
```

One main layout usually; a `PrintLayout`/`AuthLayout` for special contexts. Shared data (auth user, flash) comes from `usePage().props` (set by `HandleInertiaRequests::share` — see `laravel-conventions` inertia-react.md).

## Component tiers

### `Components/ui/` — design-system primitives
Feature-agnostic, reusable everywhere: `Button`, `Input`, `Select`, `Card`, `Modal`, `Badge`, `DataTable`, `Pagination`. Stable API, no business types in their props (accept primitives/generics). These are your equivalent of a UI kit — invest in them once.

### `Components/<feature>/` — feature components
Know about a domain type (e.g. `OrderStatus`) but are still reusable within that feature. E.g. `StatusBadge`, `OrderRow`, `AmountInput`.

### Page-local components
Used in exactly one page and nowhere else → keep them in the page file (or a sibling file) until a second page needs them. Don't pre-extract.

## When to extract a component

Extract when **any** of:
- The same JSX pattern appears in **2+ places** (Rule of Three relaxed to Two for visual duplication).
- A chunk of the page is conceptually one thing (a `OrderSummary`) and extracting it makes the page read top-to-bottom.
- The same **class string** is copy-pasted 3+ times → extract, or use a primitive that owns those classes.

Do **not** extract when:
- It's used once and the page is still readable.
- It's a styling one-off (just write the classes inline).
- You'd pass 6+ props through and the parent gets harder to read.

The goal is readability of the page, not maximizing component count.

## Props and typing

Type all component props explicitly; prefer `interface` for object props. Keep primitives primitive:

```tsx
// ui/Button.tsx
type Variant = 'primary' | 'secondary' | 'danger';
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: Variant;
}
export function Button({ variant = 'primary', className, ...rest }: ButtonProps) {
  const base = 'inline-flex items-center rounded px-4 py-2 font-medium';
  const variants: Record<Variant, string> = {
    primary: 'bg-blue-600 text-white hover:bg-blue-700',
    secondary: 'bg-gray-100 text-gray-900 hover:bg-gray-200',
    danger: 'bg-red-600 text-white hover:bg-red-700',
  };
  return <button className={`${base} ${variants[variant]} ${className ?? ''}`} {...rest} />;
}
```

Page props derive from the backend Resource types (see [types-for-jsonb.md](types-for-jsonb.md)).

## Custom hooks

Put genuinely reusable stateful logic in `Hooks/`:
- `useDebounce(value, ms)` — generic.
- `useLocalStorage(key, initial)` — generic.
- `useOrders(filters)` — feature-specific data fetching (when not using Inertia partial reloads).

A hook should encapsulate one piece of stateful behavior. If it's just a pure function, it belongs in `lib/`, not `Hooks/`.
