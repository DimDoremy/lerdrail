# Vite Build & SSR (Lerd / FrankenPHP)

## Table of Contents
- [Vite under Laravel](#vite-under-laravel)
- [Running via Lerd](#running-via-lerd)
- [laravel-vite-plugin + Inertia](#laravel-vite-plugin--inertia)
- [SSR setup](#ssr-setup)
- [Lerd FrankenPHP worker mode](#lerd-frankenphp-worker-mode)
- [Troubleshooting](#troubleshooting)

Query Vite/Laravel integration via context7 `/laravel/docs` ("vite") and `/inertiajs/docs` ("server-side rendering").

## Vite under Laravel

`laravel-vite-plugin` wires Vite into Blade via `@viteReactRefresh` / `@vite(['resources/js/app.tsx', 'resources/css/app.css'])`. The config lives in `vite.config.ts`:

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import react from '@vitejs/plugin-react';
import { resolve } from 'path';

export default defineConfig({
  plugins: [
    laravel({
      input: ['resources/css/app.css', 'resources/js/app.tsx'],
      ssr: 'resources/js/ssr.tsx',          // SSR entry (when SSR enabled)
      refresh: true,                        // refresh on backend PHP changes during dev
    }),
    react(),
  ],
  resolve: { alias: { '@': resolve(__dirname, 'resources/js') } },
});
```

## Running via Lerd

Always run npm/Vite through Lerd to use the project's pinned Node version:

```bash
lerd npm install
lerd npm run dev      # Vite dev server + HMR, auto-refreshes on PHP changes
lerd npm run build    # production build → public/build/
```

Do not run `npm`/`npx` from the host shell — it may use a different Node and produce inconsistent builds. `lerd node`/`lerd npm`/`lerd npx` use the version pinned by `lerd isolate:node`.

## laravel-vite-plugin + Inertia

The client entry hydrates the Inertia app:

```tsx
// resources/js/app.tsx
import '../css/app.css';
import { createInertiaApp } from '@inertiajs/react';
import { createRoot } from 'react-dom/client';
import { resolvePageComponent } from 'laravel-vite-plugin/inertia-helpers';

createInertiaApp({
  title: (title) => `${title} — ${appName}`,
  resolve: (name) => resolvePageComponent(
    `./Pages/${name}.tsx`,
    import.meta.glob('./Pages/**/*.tsx'),
  ),
  setup({ el, App, props }) {
    createRoot(el).render(<App {...props} />);
  },
});
```

`import.meta.glob` lets Vite code-split each page — only the visited page's JS loads.

## SSR setup

Optional but recommended for content/SEO pages. SSR renders the first paint on the server (Laravel), then hydrates client-side.

1. Add the SSR entry:
```tsx
// resources/js/ssr.tsx
import { createInertiaApp } from '@inertiajs/react';
import { renderToString } from 'react-dom/server';
import { resolvePageComponent } from 'laravel-vite-plugin/inertia-helpers';
import createServer from '@inertiajs/react/server';

createServer((page) =>
  createInertiaApp({
    page,
    render: renderToString,
    resolve: (name) => resolvePageComponent(
      `./Pages/${name}.tsx`,
      import.meta.glob('./Pages/**/*.tsx'),
    ),
    setup: ({ App, props }) => <App {...props} />,
  })
);
```
2. Point `laravel-vite-plugin` at it (`ssr: 'resources/js/ssr.tsx'`, already in the config above).
3. Build the SSR bundle: `lerd npm run build` (produces `public/build/ssr.mjs` when `ssr` is configured).
4. Enable SSR in Laravel — it will detect the built SSR bundle and use it for first paint.

## Lerd FrankenPHP worker mode

For best SSR (and general) performance under Lerd, run PHP-FPM under the **FrankenPHP** runtime in **worker mode**, which keeps the Laravel app booted in memory across requests:

```bash
lerd runtime frankenphp --worker
lerd runtime frankenphp --no-worker   # disable worker mode
lerd runtime fpm                      # back to PHP-FPM
```

Worker mode dramatically cuts per-request boot cost — valuable for SSR where each request boots React. Note: worker mode persists app state in memory, so after deploys/code changes use `lerd octane:reload on` (or restart) to pick up changes.

## Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| `npm` build differs between teammates | Not using `lerd npm` → inconsistent Node. Always `lerd npm`. |
| HMR not picking up PHP/Blade changes | Ensure `refresh: true` in `laravel-vite-plugin`. |
| SSR not kicking in | `ssr` entry missing from config; `lerd npm run build` didn't emit `ssr.mjs`. |
| Stale data after deploy under worker mode | Reload: `lerd octane:reload` (or toggle worker mode). |
| Page JS not splitting | Ensure `import.meta.glob('./Pages/**/*.tsx')`, not a single import. |
| Host `npm run dev` can't reach Vite | Run under Lerd: `lerd npm run dev`. |
