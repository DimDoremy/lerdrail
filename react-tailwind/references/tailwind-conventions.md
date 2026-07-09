# Tailwind Conventions

## Table of Contents
- [Utility-first, disciplined](#utility-first-disciplined)
- [Design tokens](#design-tokens)
- [The extraction threshold](#the-extraction-threshold)
- [Responsive: mobile-first](#responsive-mobile-first)
- [Dark mode](#dark-mode)
- [When @apply is acceptable](#when-apply-is-acceptable)
- [Common patterns](#common-patterns)

Query current Tailwind docs via context7 `/tailwindlabs/tailwindcss.com` (v4 focus).

## Utility-first, disciplined

Tailwind's premise is composing utilities instead of writing custom CSS. The discipline that keeps it readable at scale:

1. **Tokens, not magic numbers.** Spacing/colors/type come from the theme.
2. **Extract when duplication appears.** A wall of utilities copied N times is a component waiting to be born.
3. **Mobile-first ordering.** `sm:`/`md:`/`lg:` read upward.
4. **Prefer a component over `@apply`** for reusable UI; reserve `@apply` for the rare global base.

## Design tokens

Define your palette, spacing, type scale, radii, shadows once in the theme (v4: `@theme` in CSS; v3: `tailwind.config.js`). Then reference by token name in classes — never inline `text-[#3b82f6]` or `mt-[13px]` for a value that should be on the scale.

```css
/* app.css (Tailwind v4 @theme) */
@theme {
  --color-brand-500: oklch(0.62 0.19 255);
  --color-brand-600: oklch(0.55 0.2 255);
  --radius-card: 0.75rem;
}
```
```tsx
<button className="bg-brand-600 text-white rounded-card">Save</button>
```

If you reach for an arbitrary value `[…]`, ask: should this be a token? Usually yes.

## The extraction threshold

| Situation | Action |
|-----------|--------|
| A class string used once | Inline it |
| Same class string in 2 places | Extract a component primitive in `Components/ui/` |
| 3+ near-identical variants (`primary`/`secondary`/`danger`) | One component with a `variant` prop + a lookup table (see Button in component-structure.md) |
| Genuinely one-off styling | Inline, don't over-engineer |

The signal is **duplication across files**, not "the class string is long" — a long one-off is fine inline.

## Responsive: mobile-first

Write the base (mobile) styles un-prefixed, then layer breakpoints upward:

```tsx
// GOOD: mobile-first
<div className="block md:flex lg:grid grid-cols-3">

// BAD: desktop-first, confusing order
<div className="lg:grid grid-cols-3 md:flex block">
```

Breakpoints (`sm` 640, `md` 768, `lg` 1024, `xl` 1280, `2xl` 1536 by default) — use 2-3 max per element; more signals the layout should be a dedicated component.

## Dark mode

Use the `dark:` variant on a class/selector strategy. Centralize the *decision* of what inverts (surfaces, text) via tokens so dark mode is mostly automatic:

```css
@theme {
  --color-surface: var(--surface-light);
  --color-text: var(--text-light);
}
.dark {
  --color-surface: var(--surface-dark);
  --color-text: var(--text-dark);
}
```
```tsx
<div className="bg-surface text-text">...</div>   // auto-adapts via tokens
```

Prefer token-driven dark mode over sprinkling `dark:` on every utility.

## When @apply is acceptable

`@apply` is fine for:
- A **global base** reset/typography in `app.css` (`body { @apply bg-surface text-text antialiased; }`).
- A genuinely reusable non-component rule (e.g. a `.prose`-like content style).

`@apply` is a smell when:
- You could just compose utilities in JSX.
- You're hiding what would be a real component behind a class name.
- The class names get long — that's a component, extract it.

Default to composing utilities in JSX; reach for `@apply` only for the global base.

## Common patterns

**Flex row with gap:**
```tsx
<div className="flex items-center gap-3">...</div>
```

**Stack (vertical):**
```tsx
<div className="flex flex-col gap-4">...</div>
```

**Card surface:**
```tsx
<div className="rounded-card border border-gray-200 bg-white p-6 shadow-sm dark:border-gray-700 dark:bg-gray-900">...</div>
```

**Responsive grid:**
```tsx
<div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3">...</div>
```

**Truncate to N lines:**
```tsx
<p className="line-clamp-2">...</p>
```

**Button variants** — see the `variant` lookup pattern in component-structure.md; it keeps class strings DRY without `@apply`.

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Arbitrary hex/px everywhere | Use theme tokens |
| Copy-pasted 8-class strings | Extract a `Components/ui/` primitive |
| `@apply` for everything | Compose utilities in JSX; `@apply` only for global base |
| Desktop-first responsive classes | Write mobile-first, layer upward |
| Sprinkling `dark:` on every utility | Token-driven dark mode (surfaces/text as tokens) |
