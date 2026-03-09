# Performance Patterns

## Contents
- Performance profiling workflow
- Code splitting strategies
- Image optimization
- Font loading
- Caching strategies
- Rendering optimization

## Performance profiling workflow

Measure before optimizing. Gut feelings about performance are usually wrong.

### 1. Lighthouse audit

```bash
# CLI for CI integration
npx lighthouse http://localhost:5173 --output=json --output-path=./report.json

# Key thresholds
# Performance: ≥ 90
# LCP: < 2.5s
# INP: < 200ms
# CLS: < 0.1
# Total Blocking Time: < 200ms
```

### 2. Chrome DevTools Performance tab

- Record a page load or user interaction.
- Look for: long tasks (>50ms), layout thrashing, excessive re-renders.
- Flame chart: wide bars = slow functions. Drill into the call stack.

### 3. Web Vitals in code

```tsx
import { onLCP, onINP, onCLS } from 'web-vitals';

onLCP(console.log);
onINP(console.log);
onCLS(console.log);
```

Send metrics to your analytics backend in production. Lab data (Lighthouse) and field data (real users) often differ.

### 4. React DevTools Profiler

- Enable "Highlight updates when components render" to spot unnecessary re-renders.
- Record an interaction, inspect which components re-rendered and why.
- "Why did this render?" shows the changed props/state/hooks.

## Code splitting strategies

### Route-based splitting (start here)

Every route becomes its own chunk. Users only download the code for the page they visit:

```tsx
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

const Dashboard = lazy(() => import('./features/dashboard'));
const Settings = lazy(() => import('./features/settings'));
const Analytics = lazy(() => import('./features/analytics'));

function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
        <Route path="/analytics" element={<Analytics />} />
      </Routes>
    </Suspense>
  );
}
```

### Component-based splitting

Lazy-load heavy components that aren't visible on initial render:

```tsx
const MarkdownEditor = lazy(() => import('./components/MarkdownEditor'));
const ChartWidget = lazy(() => import('./components/ChartWidget'));

function PostEditor({ mode }: { mode: 'edit' | 'preview' }) {
  return (
    <Suspense fallback={<Skeleton height={400} />}>
      {mode === 'edit' ? <MarkdownEditor /> : <Preview />}
    </Suspense>
  );
}
```

### Library-based splitting

Heavy libraries (chart libs, date pickers, syntax highlighters) should load on demand:

```tsx
// Vite: manual chunk splitting for shared vendor code
// vite.config.ts
build: {
  rollupOptions: {
    output: {
      manualChunks: {
        'chart-vendor': ['chart.js', 'react-chartjs-2'],
        'editor-vendor': ['@codemirror/state', '@codemirror/view'],
      },
    },
  },
}
```

### Prefetching

Preload chunks the user is likely to navigate to:

```tsx
// Prefetch on hover/focus — loads chunk before click
function NavLink({ to, children }: { to: string; children: React.ReactNode }) {
  const prefetch = () => {
    // Dynamic import triggers chunk download without rendering
    if (to === '/analytics') import('./features/analytics');
    if (to === '/settings') import('./features/settings');
  };

  return (
    <Link to={to} onMouseEnter={prefetch} onFocus={prefetch}>
      {children}
    </Link>
  );
}
```

## Image optimization

### Format selection

| Format | Use case | Browser support |
|--------|----------|-----------------|
| AVIF | Photos, complex images. Best compression. | Chrome, Firefox, Safari 16.4+ |
| WebP | Photos, fallback from AVIF. ~30% smaller than JPEG. | All modern browsers |
| SVG | Icons, logos, illustrations. Scales without loss. | All browsers |
| PNG | Screenshots, images needing transparency with sharp edges. | All browsers |

### Responsive images with srcset

Serve the right size for the viewport. Don't send a 2000px image to a 400px container:

```html
<img
  src="/images/hero-800.webp"
  srcset="
    /images/hero-400.webp 400w,
    /images/hero-800.webp 800w,
    /images/hero-1200.webp 1200w,
    /images/hero-1600.webp 1600w
  "
  sizes="(max-width: 768px) 100vw, 50vw"
  alt="Product showcase"
  width="800"
  height="600"
  loading="lazy"
  decoding="async"
/>
```

### Lazy loading

```html
<!-- Below the fold: lazy load -->
<img src="photo.webp" loading="lazy" width="400" height="300" alt="..." />

<!-- Above the fold (hero, LCP image): eager load + preload -->
<link rel="preload" as="image" href="/images/hero.avif" type="image/avif" />
<img src="/images/hero.avif" width="1200" height="600" alt="..." fetchpriority="high" />
```

Always set `width` and `height` to prevent layout shift (CLS). The browser calculates the aspect ratio before the image loads.

## Font loading

### font-display strategy

```css
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter-var.woff2') format('woff2');
  font-display: swap;  /* show fallback immediately, swap when loaded */
  font-weight: 100 900;
}
```

- `swap`: text visible immediately with fallback font, swaps when custom font loads. Best for body text.
- `optional`: browser decides whether to use the custom font based on connection speed. Best for non-critical fonts.

### Preload critical fonts

```html
<link rel="preload" href="/fonts/inter-var.woff2" as="font" type="font/woff2" crossorigin />
```

Only preload 1-2 fonts used above the fold. Every preload competes for bandwidth.

### Subsetting

Strip unused glyphs. A full font file can be 200KB+. Latin subset is typically 15-30KB:

```bash
# Using pyftsubset (fonttools)
pyftsubset Inter.ttf \
  --output-file=inter-latin.woff2 \
  --flavor=woff2 \
  --layout-features='kern,liga' \
  --unicodes='U+0000-00FF,U+2000-206F'
```

### System font fallback stack

Minimize layout shift by matching fallback metrics:

```css
:root {
  --font-body: 'Inter', ui-sans-serif, system-ui, -apple-system, sans-serif;
}

/* Adjust fallback metrics to match Inter */
@font-face {
  font-family: 'Inter Fallback';
  src: local('Arial');
  ascent-override: 90%;
  descent-override: 22%;
  line-gap-override: 0%;
  size-adjust: 107%;
}
```

## Caching strategies

### Hashed filenames (Vite default)

```
dist/assets/index-a1b2c3d4.js    → changes on every code change
dist/assets/vendor-e5f6g7h8.js   → changes only when deps change
dist/index.html                    → never hashed, always fresh
```

### Cache-Control headers

```
# Hashed assets: cache forever (hash changes on update)
/assets/*  Cache-Control: public, max-age=31536000, immutable

# HTML: always revalidate (picks up new asset hashes)
/index.html  Cache-Control: no-cache

# API responses: short cache with revalidation
/api/*  Cache-Control: private, max-age=0, must-revalidate
```

### Service worker for offline support

```ts
// sw.ts — cache-first for assets, network-first for API
self.addEventListener('fetch', (event: FetchEvent) => {
  const { request } = event;
  if (request.url.includes('/assets/')) {
    event.respondWith(cacheFirst(request));
  } else if (request.url.includes('/api/')) {
    event.respondWith(networkFirst(request));
  }
});

async function cacheFirst(request: Request): Promise<Response> {
  const cached = await caches.match(request);
  return cached ?? fetch(request);
}

async function networkFirst(request: Request): Promise<Response> {
  try {
    const response = await fetch(request);
    const cache = await caches.open('api-v1');
    cache.put(request, response.clone());
    return response;
  } catch {
    const cached = await caches.match(request);
    return cached ?? new Response('Offline', { status: 503 });
  }
}
```

## Rendering optimization

### Virtual scrolling

Render only visible items. DOM node count stays constant regardless of list size:

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null);
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 60,
    overscan: 5,
  });

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>
        {virtualizer.getVirtualItems().map(virtualRow => (
          <div
            key={virtualRow.key}
            style={{
              position: 'absolute',
              top: 0,
              transform: `translateY(${virtualRow.start}px)`,
              height: `${virtualRow.size}px`,
              width: '100%',
            }}
          >
            <ItemRow item={items[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Memoization rules

1. **Profile first**. Don't memoize until you've confirmed a re-render problem.
2. **`React.memo`** for components that receive the same props frequently but parent re-renders often.
3. **`useMemo`** for expensive computations (filtering large arrays, complex calculations).
4. **`useCallback`** for callback props passed to memoized children.
5. **Stable references**. Objects/arrays created in render are new every time — hoist constants, memoize derived objects.

```tsx
// Bad: new object on every render, breaks memo on child
function Parent({ userId }: { userId: string }) {
  const style = { color: 'red', fontSize: 14 };
  return <MemoizedChild style={style} />;
}

// Good: stable reference
const childStyle = { color: 'red', fontSize: 14 } as const;
function Parent({ userId }: { userId: string }) {
  return <MemoizedChild style={childStyle} />;
}
```

### requestAnimationFrame for visual updates

Batch DOM reads and writes to avoid layout thrashing:

```ts
function animateProgress(element: HTMLElement, from: number, to: number) {
  let current = from;
  function step() {
    current += (to - current) * 0.1;
    element.style.width = `${current}%`;
    if (Math.abs(to - current) > 0.5) {
      requestAnimationFrame(step);
    }
  }
  requestAnimationFrame(step);
}
```

### Web Workers for heavy computation

Move CPU-intensive work off the main thread:

```ts
// worker.ts
self.addEventListener('message', (e: MessageEvent<{ data: number[] }>) => {
  const result = heavyComputation(e.data.data);
  self.postMessage(result);
});

// component.tsx
const worker = new Worker(new URL('./worker.ts', import.meta.url), { type: 'module' });

function useWorkerComputation(data: number[]) {
  const [result, setResult] = useState<number | null>(null);

  useEffect(() => {
    worker.postMessage({ data });
    worker.onmessage = (e) => setResult(e.data);
  }, [data]);

  return result;
}
```