# Component Patterns

## Contents
- Feature-based directory structure
- Composition patterns
- Form handling
- Error boundaries
- Compound components

## Feature-based directory structure

Group by feature, not by type. Co-locate everything a feature needs:

```
src/
├── features/
│   ├── auth/
│   │   ├── components/
│   │   │   ├── LoginForm.tsx
│   │   │   ├── LoginForm.module.css
│   │   │   └── LoginForm.test.tsx
│   │   ├── hooks/
│   │   │   └── useAuth.ts
│   │   ├── api/
│   │   │   └── auth.ts
│   │   ├── types.ts
│   │   └── index.ts          # public API barrel
│   ├── dashboard/
│   │   ├── components/
│   │   ├── hooks/
│   │   └── index.ts
│   └── settings/
├── shared/
│   ├── components/            # Button, Modal, Input — truly reusable
│   ├── hooks/                 # useDebounce, useMediaQuery
│   ├── utils/                 # formatDate, cn()
│   └── types/                 # global type definitions
├── app/
│   ├── routes.tsx
│   ├── providers.tsx
│   └── App.tsx
└── main.tsx
```

The barrel `index.ts` exports the public surface. Other features import from the barrel, never from internal paths:

```tsx
// features/auth/index.ts
export { LoginForm } from './components/LoginForm';
export { useAuth } from './hooks/useAuth';
export type { User, AuthState } from './types';
```

## Composition patterns

### Children and slots

Pass components as props for flexible layouts:

```tsx
interface PageLayoutProps {
  header: React.ReactNode;
  sidebar?: React.ReactNode;
  children: React.ReactNode;
}

function PageLayout({ header, sidebar, children }: PageLayoutProps) {
  return (
    <div className={styles.layout}>
      <header className={styles.header}>{header}</header>
      <div className={styles.body}>
        {sidebar && <aside className={styles.sidebar}>{sidebar}</aside>}
        <main className={styles.main}>{children}</main>
      </div>
    </div>
  );
}
```

### Custom hooks for logic extraction

When a component's logic outgrows its render, extract a hook:

```tsx
function useInfiniteList<T>(fetchFn: (page: number) => Promise<T[]>) {
  const [items, setItems] = useState<T[]>([]);
  const [page, setPage] = useState(1);
  const [hasMore, setHasMore] = useState(true);

  const loadMore = useCallback(async () => {
    const next = await fetchFn(page);
    setItems(prev => [...prev, ...next]);
    setHasMore(next.length > 0);
    setPage(prev => prev + 1);
  }, [fetchFn, page]);

  return { items, loadMore, hasMore };
}
```

The component stays small — it renders, the hook manages state:

```tsx
function ProductList() {
  const { items, loadMore, hasMore } = useInfiniteList(fetchProducts);
  return (
    <>
      <ul>{items.map(p => <ProductCard key={p.id} product={p} />)}</ul>
      {hasMore && <button onClick={loadMore}>Load more</button>}
    </>
  );
}
```

## Form handling

### Controlled inputs with schema validation

Use Zod for validation, keep error display accessible:

```tsx
import { z } from 'zod';

const contactSchema = z.object({
  name: z.string().min(1, 'Name is required'),
  email: z.string().email('Invalid email address'),
  message: z.string().min(10, 'Message must be at least 10 characters'),
});

type ContactForm = z.infer<typeof contactSchema>;

function ContactForm() {
  const [values, setValues] = useState<ContactForm>({
    name: '', email: '', message: '',
  });
  const [errors, setErrors] = useState<Partial<Record<keyof ContactForm, string>>>({});

  function handleChange(field: keyof ContactForm, value: string) {
    setValues(prev => ({ ...prev, [field]: value }));
    setErrors(prev => ({ ...prev, [field]: undefined }));
  }

  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    const result = contactSchema.safeParse(values);
    if (!result.success) {
      const fieldErrors: typeof errors = {};
      result.error.issues.forEach(issue => {
        const field = issue.path[0] as keyof ContactForm;
        fieldErrors[field] = issue.message;
      });
      setErrors(fieldErrors);
      return;
    }
    submitContact(result.data);
  }

  return (
    <form onSubmit={handleSubmit} noValidate>
      <FormField
        label="Name"
        name="name"
        value={values.name}
        error={errors.name}
        onChange={v => handleChange('name', v)}
      />
      <FormField
        label="Email"
        name="email"
        type="email"
        value={values.email}
        error={errors.email}
        onChange={v => handleChange('email', v)}
      />
      <FormField
        label="Message"
        name="message"
        as="textarea"
        value={values.message}
        error={errors.message}
        onChange={v => handleChange('message', v)}
      />
      <button type="submit">Send</button>
    </form>
  );
}
```

### Accessible form field

Link label, input, and error message with `id`, `htmlFor`, `aria-describedby`, and `aria-invalid`:

```tsx
interface FormFieldProps {
  label: string;
  name: string;
  value: string;
  error?: string;
  type?: string;
  as?: 'input' | 'textarea';
  onChange: (value: string) => void;
}

function FormField({ label, name, value, error, type = 'text', as = 'input', onChange }: FormFieldProps) {
  const inputId = `field-${name}`;
  const errorId = `${inputId}-error`;
  const Component = as;

  return (
    <div>
      <label htmlFor={inputId}>{label}</label>
      <Component
        id={inputId}
        name={name}
        type={as === 'input' ? type : undefined}
        value={value}
        onChange={e => onChange(e.target.value)}
        aria-invalid={!!error}
        aria-describedby={error ? errorId : undefined}
      />
      {error && <span id={errorId} role="alert">{error}</span>}
    </div>
  );
}
```

## Error boundaries

Catch rendering errors, show fallback UI, report to monitoring:

```tsx
import { Component, type ErrorInfo, type ReactNode } from 'react';

interface ErrorBoundaryProps {
  fallback?: ReactNode | ((error: Error) => ReactNode);
  onError?: (error: Error, info: ErrorInfo) => void;
  children: ReactNode;
}

interface ErrorBoundaryState {
  error: Error | null;
}

class ErrorBoundary extends Component<ErrorBoundaryProps, ErrorBoundaryState> {
  state: ErrorBoundaryState = { error: null };

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { error };
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    this.props.onError?.(error, info);
  }

  render() {
    if (this.state.error) {
      const { fallback } = this.props;
      if (typeof fallback === 'function') return fallback(this.state.error);
      return fallback ?? <DefaultErrorFallback error={this.state.error} />;
    }
    return this.props.children;
  }
}

function DefaultErrorFallback({ error }: { error: Error }) {
  return (
    <div role="alert">
      <h2>Something went wrong</h2>
      <pre>{error.message}</pre>
      <button onClick={() => window.location.reload()}>Reload page</button>
    </div>
  );
}
```

Usage — nest boundaries at different granularity levels:

```tsx
// App-level: catches routing and top-level errors
<ErrorBoundary onError={reportToSentry}>
  <RouterProvider router={router} />
</ErrorBoundary>

// Feature-level: isolates failures to one section
<ErrorBoundary fallback={<p>Failed to load dashboard widget.</p>}>
  <RevenueChart />
</ErrorBoundary>
```

## Compound components

Share implicit state between related components via context. Users compose them freely without prop threading:

```tsx
import { createContext, useContext, useState, type ReactNode } from 'react';

// --- Context ---
interface TabsContextValue {
  activeTab: string;
  setActiveTab: (id: string) => void;
}

const TabsContext = createContext<TabsContextValue | null>(null);

function useTabsContext() {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error('Tab components must be used within <Tabs>');
  return ctx;
}

// --- Tabs root ---
interface TabsProps {
  defaultTab: string;
  children: ReactNode;
}

function Tabs({ defaultTab, children }: TabsProps) {
  const [activeTab, setActiveTab] = useState(defaultTab);
  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <div>{children}</div>
    </TabsContext.Provider>
  );
}

// --- TabList ---
function TabList({ children }: { children: ReactNode }) {
  return <div role="tablist">{children}</div>;
}

// --- Tab trigger ---
function Tab({ id, children }: { id: string; children: ReactNode }) {
  const { activeTab, setActiveTab } = useTabsContext();
  return (
    <button
      role="tab"
      aria-selected={activeTab === id}
      aria-controls={`panel-${id}`}
      id={`tab-${id}`}
      onClick={() => setActiveTab(id)}
    >
      {children}
    </button>
  );
}

// --- Tab panel ---
function TabPanel({ id, children }: { id: string; children: ReactNode }) {
  const { activeTab } = useTabsContext();
  if (activeTab !== id) return null;
  return (
    <div
      role="tabpanel"
      id={`panel-${id}`}
      aria-labelledby={`tab-${id}`}
    >
      {children}
    </div>
  );
}

// --- Attach sub-components ---
Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;
```

Usage — clean, declarative API:

```tsx
<Tabs defaultTab="overview">
  <Tabs.List>
    <Tabs.Tab id="overview">Overview</Tabs.Tab>
    <Tabs.Tab id="analytics">Analytics</Tabs.Tab>
    <Tabs.Tab id="settings">Settings</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel id="overview"><OverviewContent /></Tabs.Panel>
  <Tabs.Panel id="analytics"><AnalyticsContent /></Tabs.Panel>
  <Tabs.Panel id="settings"><SettingsContent /></Tabs.Panel>
</Tabs>
```

## Internationalization (i18n)

Use `react-i18next` (most popular) or `react-intl` (ICU message format).

### Setup with react-i18next

```tsx
// i18n.ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import Backend from 'i18next-http-backend';
import LanguageDetector from 'i18next-browser-languagedetector';

i18n
  .use(Backend)            // load translations from /locales/{lng}/{ns}.json
  .use(LanguageDetector)   // detect user language
  .use(initReactI18next)
  .init({
    fallbackLng: 'en',
    ns: ['common', 'auth', 'dashboard'],
    defaultNS: 'common',
    interpolation: { escapeValue: false }, // React handles XSS
  });
```

### Usage in components

```tsx
import { useTranslation } from 'react-i18next';

function OrderSummary({ count, total }: { count: number; total: number }) {
  const { t } = useTranslation('orders');
  return (
    <p>{t('summary', { count, total: total.toFixed(2) })}</p>
  );
}
```

```json
// locales/en/orders.json
{
  "summary_one": "{{count}} item — ${{total}}",
  "summary_other": "{{count}} items — ${{total}}"
}
```

### Guidelines

- Keep translation keys semantic (`auth.login_button`), not tied to English text.
- Use ICU pluralization for counts — don't build plural logic in code.
- Extract translations with `i18next-parser` in CI to catch missing keys.
- Load translations lazily per namespace to avoid bloating the initial bundle.
