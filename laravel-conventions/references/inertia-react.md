# Inertia.js Integration (controller → props → React page)

## Table of Contents
- [The flow](#the-flow)
- [Rendering](#rendering)
- [Props & resources](#props--resources)
- [Shared data](#shared-data)
- [Partial reloads & prefetching](#partial-reloads--prefetching)
- [Forms (useForm)](#forms-useform)
- [Server-side rendering](#server-side-rendering)
- [Routing](#routing)

Query Inertia docs via context7 `/inertiajs/docs` + `/inertiajs/inertia-laravel`. React page/component conventions are in `react-tailwind`.

## The flow

```
Request → Route → Controller
   → Form Request validates
   → Controller loads models, dispatches jobs/events
   → API Resource shapes the output
   → Inertia::render('Page', $props) returns an Inertia response
   → client mounts the React page with those props (SPA-style, no page reload)
```

This project uses Inertia.js + React (single codebase; Laravel owns routing/controllers/props, React owns the rendered components). No separate API or SPA router.

## Rendering

```php
// route
Route::get('/orders/{order}', [OrderController::class, 'show'])->name('orders.show');

// controller
class OrderController extends Controller
{
    public function show(Request $request, Order $order): Response
    {
        $order->load(['user', 'items']);
        return Inertia::render('Orders/Show', [
            'order' => OrderResource::make($order),
        ]);
    }
}
```

`Inertia::render($component, $props)` — `$component` maps to a React file under `resources/js/Pages` (e.g. `'Orders/Show'` → `Pages/Orders/Show.tsx`).

## Props & resources

**Always shape props through an API Resource** — don't pass raw Eloquent models. It gives a stable, explicit contract to the React side and centralizes what's exposed.

```php
'orders' => OrderResource::collection($orders),   // list
'order'  => OrderResource::make($order),           // single
```

For props that are **expensive** and only needed sometimes, mark them lazy:

```php
return Inertia::render('Orders/Index', [
    'orders' => fn () => OrderResource::collection(Order::paginate()),   // closure = always-evaluated
    'filters'=> Inertia::lazy(fn () => $request->only('status','q')),   // only when requested via `only`
    'export' => Inertia::lazy(fn () => $this->buildExport($request)),   // computed only when client asks
]);
```

`Inertia::lazy` props are evaluated only when the client requests them by name in a partial reload — keeps initial loads fast.

## Shared data

Data every page needs (auth user, flash messages, feature flags) — share globally instead of repeating in each controller:

```php
// app/Http/Middleware/HandleInertiaRequests
public function share(Request $request): array
{
    return array_merge(parent::share($request), [
        'auth'  => ['user' => $request->user() ? UserResource::make($request->user()) : null],
        'flash' => ['success' => $request->session()->get('success')],
        'ziggy' => fn () => [...],   // route names for the client, if using Ziggy
    ]);
}
```

Accessed on the React side via `usePage().props.auth.user` etc.

## Partial reloads & prefetching

Keep navigation cheap — request only the props that changed:

```jsx
import { router } from '@inertiajs/react';
// re-fetch only the 'orders' prop, keep the rest of the page
router.get('/orders', { status: 'paid' }, { only: ['orders'] });
```

Prefetch links for snappy navigation:
```jsx
<Link href="/orders" prefetch="mount">Orders</Link>      {/* prefetch on mount */}
<Link href="/orders" prefetch="click">Orders</Link>      {/* prefetch on hover/click */}
```

## Forms (useForm)

The `@inertiajs/react` `useForm` helper handles state, submission, validation errors, and progress. This is the default form mechanism — avoid reinventing it with manual fetch + useState.

```jsx
import { useForm } from '@inertiajs/react';

function CreateOrder() {
  const { data, setData, post, processing, errors, reset } = useForm({
    number: '',
    amount_cents: '',
  });

  function submit(e) {
    e.preventDefault();
    post('/orders', { onSuccess: () => reset() });
  }

  return (
    <form onSubmit={submit}>
      <input value={data.number} onChange={e => setData('number', e.target.value)} />
      {errors.number && <span>{errors.number}</span>}
      <input value={data.amount_cents} onChange={e => setData('amount_cents', e.target.value)} />
      {errors.amount_cents && <span>{errors.amount_cents}</span>}
      <button type="submit" disabled={processing}>Create</button>
    </form>
  );
}
```

`useForm` returns `{ data, setData, transform, reset, clearErrors, setError, defaults, isDirty, processing, wasSuccessful, recentlySuccessful, progress, errors, hasErrors, post/get/put/delete/submit, optimistic }`. Validation errors come back from the server's Form Request and populate `errors` automatically. For optimistic updates use `.optimistic(props => ({...})).post(...)`.

The `<Form>` component (render-prop slot) exposes the same state via a function-as-child for declarative forms.

## Server-side rendering

Optional. When enabled, Inertia renders the React page to HTML on the server (Laravel) for first paint, then hydrates. Under Lerd, run SSR via the FrankenPHP worker runtime:

```bash
lerd runtime frankenphp --worker   # worker mode keeps the app booted
```

Set up `resources/js/ssr.tsx` as the SSR entry, configure `@vitejs/plugin-react` + `@inertiajs/react` SSR, and Laravel will dispatch SSR when the Vite dev server / build output provides it. See `react-tailwind`'s vite-ssr reference and context7 `/inertiajs/docs` "server-side rendering".

## Routing

Routing stays server-side (this is the Inertia point — classic routes + controllers, SPA feel). Use named routes; expose route URLs to the client via Ziggy if you need to generate them in React. Always validate on the server (Form Requests) regardless of client-side UX validation.

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Passing raw Eloquent to `Inertia::render` | Wrap in an API Resource |
| Re-fetching the whole page when a filter changes | `only: ['orders']` partial reload |
| Manual `fetch`+`useState` for a form | Use `useForm` |
| Repeating auth/flash in every controller | Share globally via `HandleInertiaRequests::share` |
| Client-side-only validation | Always validate in the Form Request too |
