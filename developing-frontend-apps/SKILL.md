---
name: developing-frontend-apps
description: Frontend application development best practices. Use when building, modifying, or reviewing frontend applications, UI components, client-side JavaScript/TypeScript, CSS/styling, or web application architecture.
---

# Frontend Application Development Best Practices

## Component Architecture

- One component, one job. If it fetches data AND renders a complex UI, split it.
- Composition over inheritance — use children, render props, and custom hooks.
- Presentational components receive data via props and emit events. Container components handle data fetching and state.
- Co-locate related files — component, styles, tests, types in the same directory.
- Define an explicit Props interface for every component:
  ```tsx
  interface UserCardProps {
    user: User;
    onEdit: (id: string) => void;
    variant?: 'compact' | 'full';
  }

  function UserCard({ user, onEdit, variant = 'full' }: UserCardProps) {
    // ...
  }
  ```
- Avoid prop drilling beyond 2 levels — use context or composition (pass components, not data).
- Keep components under ~200 lines. Extract hooks when logic dominates the render.

## State Management

- **Local state first**. `useState` or `useReducer` for component-scoped concerns.
- **Lift state only when siblings share it**. Move to the nearest common ancestor, no higher.
- **Server state is not app state**. Use a data-fetching library (TanStack Query, SWR) — it handles caching, refetching, optimistic updates.
- **Minimal global state**. Reserve for truly app-wide concerns: auth, theme, locale.
- **Derive, don't duplicate**. Compute values from source state:
  ```tsx
  // Bad: syncing filtered list into separate state
  const [items, setItems] = useState<Item[]>([]);
  const [filtered, setFiltered] = useState<Item[]>([]);

  // Good: derive from source
  const filtered = useMemo(
    () => items.filter(i => i.status === activeFilter),
    [items, activeFilter]
  );
  ```
- **Global state libraries** (when context isn't enough):
  - **Zustand** — minimal API, great for simple global state (auth, UI toggles). No boilerplate.
  - **Jotai** — atomic state, bottom-up approach. Good for independent pieces of state that compose.
  - **Redux Toolkit** — full-featured, middleware, devtools. Use for complex state with many interdependent slices.
  - Pick Zustand by default. Reach for Redux Toolkit only when you need middleware, time-travel debugging, or complex normalized state.
- **Forms**: use React Hook Form or TanStack Form for multi-field forms with validation. Manual `useState` per field doesn't scale past 3-4 fields — validation, dirty tracking, and error display become unwieldy.
- **Immutable updates**. Spread or `structuredClone` — never mutate state directly.
- **URL as state**. Search params, filters, pagination belong in the URL for shareability and back-button support.

## Performance

### Bundle size
- Tree-shake — use ES modules, avoid barrel files that pull entire libraries.
- Code-split at route boundaries with `lazy()`. Lazy-load heavy components (editors, charts, maps).
- Run `npx vite-bundle-visualizer` or `source-map-explorer` to find bloat.
- Target: initial JS payload under 200KB gzipped.

### Core Web Vitals
- **LCP < 2.5s** — preload hero image, inline critical CSS, avoid render-blocking scripts.
- **INP < 200ms** — keep main thread free, defer non-critical work, use `startTransition` for expensive updates.
- **CLS < 0.1** — set explicit dimensions on images/video, reserve space for dynamic content, avoid layout shifts from web fonts.

### Rendering
- Virtualize long lists (TanStack Virtual, react-window) — never render 1000+ DOM nodes.
- Memoize expensive components with `React.memo` and stable callback references with `useCallback`. Profile first — premature memoization adds complexity without measurable gain.
- Lazy-load images with `loading="lazy"` and always set `width`/`height` attributes.

## Accessibility

- **Semantic HTML**. `<button>` not `<div onClick>`. Use `<nav>`, `<main>`, `<aside>`, `<section>`, `<header>`, `<footer>`.
- **Keyboard operable**. Every interactive element reachable via Tab. Custom widgets need arrow key navigation.
- **Alt text**. Descriptive for informational images. Empty `alt=""` for decorative images.
- **Form labels**. Every input needs a `<label>`. Link error messages with `aria-describedby`:
  ```tsx
  <label htmlFor="email">Email</label>
  <input id="email" aria-describedby="email-error" aria-invalid={!!error} />
  {error && <span id="email-error" role="alert">{error}</span>}
  ```
- **ARIA only when HTML falls short**. A `<button>` already has `role="button"` — don't add it again.
- **Color contrast**. 4.5:1 for normal text, 3:1 for large text (18px+ bold or 24px+).
- **Focus management**. Trap focus in modals. Restore focus to the trigger element on close.
- **Test with a screen reader**. VoiceOver (macOS), NVDA (Windows). Run `axe-core` in CI.

## CSS Architecture

- **Scoped styles**. CSS Modules (`.module.css`) or Tailwind utility classes. Avoid global stylesheets beyond reset/tokens.
- **Design tokens**. Define colors, spacing, typography as CSS custom properties on `:root`:
  ```css
  :root {
    --color-primary: oklch(55% 0.25 260);
    --space-sm: 0.5rem;
    --space-md: 1rem;
    --radius-md: 0.5rem;
    --font-body: system-ui, sans-serif;
  }
  ```
- **Mobile-first**. Base styles for small screens, `@media (min-width: ...)` for larger.
- **Logical properties**. `margin-inline`, `padding-block`, `inline-size` instead of directional properties — supports RTL layouts.
- **No magic numbers**. Use tokens, `em`/`rem`, or `calc()`. Every value should have a reason.
- **Prefer gap**. `gap` on flex/grid replaces margin hacks and adjacent sibling selectors.

## TypeScript for Frontend

- **Strict mode**. Enable `strict: true` in `tsconfig.json`. No exceptions.
- **Props and state interfaces**. Define them explicitly — never inline complex types:
  ```tsx
  interface SearchState {
    query: string;
    results: SearchResult[];
    status: 'idle' | 'loading' | 'error' | 'success';
  }
  ```
- **Avoid `any`**. Use `unknown` and narrow with type guards:
  ```tsx
  function isApiError(err: unknown): err is ApiError {
    return typeof err === 'object' && err !== null && 'code' in err;
  }
  ```
- **API response types**. Generate from OpenAPI spec (`openapi-typescript`) or validate at the boundary with Zod. Never trust runtime data matches your types.
- **Discriminated unions** for state machines:
  ```tsx
  type AsyncState<T> =
    | { status: 'idle' }
    | { status: 'loading' }
    | { status: 'error'; error: Error }
    | { status: 'success'; data: T };
  ```
- **`as const`** over `enum`. Enums emit runtime code and have quirky behavior:
  ```tsx
  const ROLES = ['admin', 'editor', 'viewer'] as const;
  type Role = (typeof ROLES)[number]; // 'admin' | 'editor' | 'viewer'
  ```

## Testing

### Unit tests
- Test behavior, not implementation. Interact like a user — click, type, assert visible output.
- Query by role, label, text — not by class name or test ID (last resort).
- Mock external dependencies (API, router, storage), not internal modules.

### Integration tests
- Render full pages with mocked API (MSW). Test routing between pages, multi-step form flows, error states.
- Use a custom `render` that wraps providers (router, query client, theme).

### E2E tests
- Cover critical user paths: sign up, core workflow, payment. Keep the suite small (<50 tests) and fast (<5 minutes).
- Use Playwright. Page Object Model for reusable selectors.
- Run in CI against a staging environment or docker-compose stack.

## Security

- **XSS**. Never use `dangerouslySetInnerHTML` with user input. Sanitize with DOMPurify if you must render HTML.
- **CSP**. Set `Content-Security-Policy` header. At minimum: `default-src 'self'; script-src 'self'`.
- **CORS**. Configure on the server, not the client. Never use `Access-Control-Allow-Origin: *` with credentials.
- **Tokens**. Store in `httpOnly` cookies, not `localStorage`. `localStorage` is readable by any script on the page.
- **Dependencies**. Run `npm audit` regularly. Use `npm audit --omit=dev` for production deps. Automate with Dependabot or Renovate.
- **SRI**. Add `integrity` attribute to CDN `<script>` and `<link>` tags.
- **Error monitoring**. Use Sentry or Datadog RUM to capture client-side errors in production. Configure source maps for readable stack traces.

## Build Tooling

- **Vite** for new projects. Fast dev server (native ESM), optimized production builds (Rollup).
- **Path aliases**. `"@/*": ["./src/*"]` in `tsconfig.json` and `vite.config.ts`:
  ```ts
  // vite.config.ts
  resolve: {
    alias: { '@': path.resolve(__dirname, 'src') }
  }
  ```
- **Env vars**. `.env` files with `VITE_` prefix for client-exposed variables. Never expose secrets — `VITE_` vars are embedded in the bundle.
- **Hashed filenames**. Vite does this by default — enables aggressive caching.
- **Source maps**. Enable in production for error tracking (Sentry, Datadog). Upload maps privately, don't serve them publicly.
- **CI pipeline**: `lint` → `type-check` → `test` → `build` → `lighthouse`

## SEO Basics

- **SSR/SSG** for content that needs indexing. SPAs with client-side rendering are invisible to most crawlers.
- **Unique `<title>` and `<meta name="description">`** per page. Title under 60 chars, description under 155.
- **Open Graph**. `og:title`, `og:description`, `og:image`, `og:url` for social previews.
- **Canonical URL**. `<link rel="canonical" href="...">` to avoid duplicate content.
- **JSON-LD**. Structured data for rich results:
  ```html
  <script type="application/ld+json">
  { "@context": "https://schema.org", "@type": "Article", "headline": "..." }
  </script>
  ```
- **`robots.txt` + sitemap.xml**. Sitemap lists all indexable URLs. Submit to Google Search Console.

## New frontend app workflow

```
- [ ] Scaffold with Vite (React + TypeScript template)
- [ ] Configure strict tsconfig, path aliases, ESLint, Prettier
- [ ] Set up CSS strategy (CSS Modules or Tailwind)
- [ ] Define design tokens (CSS custom properties)
- [ ] Set up routing (React Router, TanStack Router)
- [ ] Configure data fetching (TanStack Query)
- [ ] Add testing stack (Vitest + Testing Library + MSW + Playwright)
- [ ] Add error boundary at app root
- [ ] Set up CI pipeline (lint → type-check → test → build → lighthouse)
- [ ] Configure env vars, source maps, bundle analysis
- [ ] Run validation loop (below)
```

## Validation loop

1. `npx eslint .` — fix all warnings and errors
2. `npx tsc --noEmit` — fix type errors
3. `npx vitest run` — fix failing tests
4. `npx playwright test` — fix E2E failures
5. `npx axe-core` or `jest-axe` — fix accessibility violations
6. `npx vite build && npx vite-bundle-visualizer` — verify bundle under 200KB gzipped
7. Lighthouse CI — verify performance score ≥ 90, accessibility ≥ 95
8. Repeat until all checks pass clean

## Deep-dive references

**Component patterns**: See [patterns/component-patterns.md](patterns/component-patterns.md) for directory structure, composition, forms, error boundaries, compound components
**Performance patterns**: See [patterns/performance-patterns.md](patterns/performance-patterns.md) for profiling, code splitting, images, fonts, caching, rendering optimization
**Testing patterns**: See [patterns/testing-patterns.md](patterns/testing-patterns.md) for Vitest setup, component tests, MSW mocking, Playwright E2E
**Accessibility**: See [accessibility-cheatsheet.md](accessibility-cheatsheet.md) for WCAG checklist, semantic HTML, ARIA reference, keyboard patterns

## Official references

- [WCAG 2.2](https://www.w3.org/TR/WCAG22/) — Web Content Accessibility Guidelines, Level AA target
- [web.dev](https://web.dev/) — Core Web Vitals, performance, best practices
- [Testing Library](https://testing-library.com/docs/) — query priorities, best practices, framework integrations
- [Vite](https://vite.dev/guide/) — configuration, plugins, build optimization