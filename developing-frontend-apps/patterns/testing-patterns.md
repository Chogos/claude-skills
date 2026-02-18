# Testing Patterns

## Contents
- Test setup
- Component testing
- API mocking with MSW
- Accessibility testing
- E2E with Playwright

## Test setup

### Vitest configuration

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import path from 'node:path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: { '@': path.resolve(__dirname, 'src') },
  },
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./src/test/setup.ts'],
    include: ['src/**/*.test.{ts,tsx}'],
    coverage: {
      provider: 'v8',
      include: ['src/**/*.{ts,tsx}'],
      exclude: ['src/test/**', '**/*.d.ts', '**/types.ts'],
    },
  },
});
```

### Test setup file

```ts
// src/test/setup.ts
import '@testing-library/jest-dom/vitest';
import { cleanup } from '@testing-library/react';
import { afterEach } from 'vitest';
import { server } from './mocks/server';

// MSW server lifecycle
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => {
  server.resetHandlers();
  cleanup();
});
afterAll(() => server.close());
```

### Custom render with providers

Wrap every test render with the same providers the app uses. Avoid duplicating setup in each test:

```tsx
// src/test/render.tsx
import { render, type RenderOptions } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { MemoryRouter } from 'react-router-dom';
import type { ReactElement, ReactNode } from 'react';

interface CustomRenderOptions extends Omit<RenderOptions, 'wrapper'> {
  initialEntries?: string[];
  queryClient?: QueryClient;
}

function createTestQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: { retry: false, gcTime: 0 },
      mutations: { retry: false },
    },
  });
}

function AllProviders({
  children,
  initialEntries = ['/'],
  queryClient,
}: {
  children: ReactNode;
  initialEntries?: string[];
  queryClient?: QueryClient;
}) {
  const client = queryClient ?? createTestQueryClient();
  return (
    <QueryClientProvider client={client}>
      <MemoryRouter initialEntries={initialEntries}>
        {children}
      </MemoryRouter>
    </QueryClientProvider>
  );
}

function customRender(ui: ReactElement, options: CustomRenderOptions = {}) {
  const { initialEntries, queryClient, ...renderOptions } = options;
  return render(ui, {
    wrapper: ({ children }) => (
      <AllProviders initialEntries={initialEntries} queryClient={queryClient}>
        {children}
      </AllProviders>
    ),
    ...renderOptions,
  });
}

export { customRender as render };
export { screen, within, waitFor } from '@testing-library/react';
export { default as userEvent } from '@testing-library/user-event';
```

## Component testing

### Render, interact, assert

```tsx
import { render, screen, userEvent } from '@/test/render';
import { Counter } from './Counter';

test('increments count on button click', async () => {
  const user = userEvent.setup();
  render(<Counter initialCount={0} />);

  expect(screen.getByText('Count: 0')).toBeInTheDocument();

  await user.click(screen.getByRole('button', { name: /increment/i }));

  expect(screen.getByText('Count: 1')).toBeInTheDocument();
});
```

### Query priority

Use the most accessible query available. Fallback order:

1. **`getByRole`** — how assistive tech sees the element. Preferred for buttons, links, headings, inputs.
2. **`getByLabelText`** — form fields with associated labels.
3. **`getByPlaceholderText`** — only if no label exists (fix the component instead).
4. **`getByText`** — non-interactive elements with visible text.
5. **`getByTestId`** — last resort. Means the element has no accessible name or visible text.

```tsx
// Good: queries by accessible role and name
screen.getByRole('button', { name: /submit/i });
screen.getByRole('heading', { level: 1 });
screen.getByRole('textbox', { name: /email/i });
screen.getByRole('checkbox', { name: /agree to terms/i });

// Avoid: queries by implementation detail
screen.getByTestId('submit-btn');
```

### Async operations with findBy

`findBy*` queries wait for the element to appear. Use for data that loads asynchronously:

```tsx
test('displays user profile after loading', async () => {
  render(<UserProfile userId="123" />);

  // Shows loading state first
  expect(screen.getByText(/loading/i)).toBeInTheDocument();

  // Waits for async data to render
  expect(await screen.findByRole('heading', { name: /alice/i })).toBeInTheDocument();
  expect(screen.queryByText(/loading/i)).not.toBeInTheDocument();
});
```

### Testing user interactions

Use `@testing-library/user-event` over `fireEvent`. It simulates real browser behavior (focus, keydown, keyup, input, blur):

```tsx
test('search filters results as user types', async () => {
  const user = userEvent.setup();
  render(<SearchableList items={mockItems} />);

  const search = screen.getByRole('searchbox', { name: /search/i });
  await user.type(search, 'react');

  expect(screen.getByText('React Testing')).toBeInTheDocument();
  expect(screen.queryByText('Vue Guide')).not.toBeInTheDocument();
});

test('form submission with keyboard', async () => {
  const onSubmit = vi.fn();
  const user = userEvent.setup();
  render(<LoginForm onSubmit={onSubmit} />);

  await user.type(screen.getByLabelText(/email/i), 'test@example.com');
  await user.type(screen.getByLabelText(/password/i), 'secret123');
  await user.keyboard('{Enter}');

  expect(onSubmit).toHaveBeenCalledWith({
    email: 'test@example.com',
    password: 'secret123',
  });
});
```

## API mocking with MSW

### Handler definitions

```ts
// src/test/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('/api/users/:id', ({ params }) => {
    return HttpResponse.json({
      id: params.id,
      name: 'Alice',
      email: 'alice@example.com',
    });
  }),

  http.get('/api/users', ({ request }) => {
    const url = new URL(request.url);
    const page = Number(url.searchParams.get('page') ?? '1');
    return HttpResponse.json({
      users: [{ id: '1', name: 'Alice' }, { id: '2', name: 'Bob' }],
      page,
      totalPages: 5,
    });
  }),

  http.post('/api/users', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ id: '3', ...body }, { status: 201 });
  }),
];
```

### Server setup

```ts
// src/test/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
```

### Per-test overrides

Override default handlers for error states and edge cases:

```tsx
import { http, HttpResponse } from 'msw';
import { server } from '@/test/mocks/server';

test('displays error message when API fails', async () => {
  server.use(
    http.get('/api/users/:id', () => {
      return HttpResponse.json(
        { message: 'User not found' },
        { status: 404 },
      );
    }),
  );

  render(<UserProfile userId="999" />);

  expect(await screen.findByRole('alert')).toHaveTextContent(/user not found/i);
});

test('handles network error', async () => {
  server.use(
    http.get('/api/users/:id', () => {
      return HttpResponse.error();
    }),
  );

  render(<UserProfile userId="123" />);

  expect(await screen.findByRole('alert')).toHaveTextContent(/something went wrong/i);
});
```

## Accessibility testing

### ESLint plugin

Catch accessibility issues at lint time:

```bash
npm install -D eslint-plugin-jsx-a11y
```

```js
// eslint.config.js
import jsxA11y from 'eslint-plugin-jsx-a11y';

export default [
  jsxA11y.flatConfigs.recommended,
];
```

Catches: missing alt text, invalid ARIA attributes, non-interactive elements with click handlers, missing form labels.

### axe-core integration with Vitest

```ts
// src/test/a11y.ts
import { axe, toHaveNoViolations } from 'jest-axe';
import type { RenderResult } from '@testing-library/react';

expect.extend(toHaveNoViolations);

export async function expectNoA11yViolations(container: RenderResult['container']) {
  const results = await axe(container);
  expect(results).toHaveNoViolations();
}
```

```tsx
import { render, screen } from '@/test/render';
import { expectNoA11yViolations } from '@/test/a11y';
import { LoginForm } from './LoginForm';

test('login form has no accessibility violations', async () => {
  const { container } = render(<LoginForm onSubmit={vi.fn()} />);
  await expectNoA11yViolations(container);
});
```

Run axe checks on all major pages. Add to CI — accessibility regressions caught before merge.

## E2E with Playwright

### Configuration

```ts
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  timeout: 30_000,
  retries: process.env.CI ? 2 : 0,
  use: {
    baseURL: 'http://localhost:5173',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'mobile', use: { ...devices['iPhone 14'] } },
  ],
  webServer: {
    command: 'npm run dev',
    port: 5173,
    reuseExistingServer: !process.env.CI,
  },
});
```

### Page Object Model

Encapsulate page-specific selectors and actions. Tests read like user stories:

```ts
// e2e/pages/login.page.ts
import { type Page, type Locator } from '@playwright/test';

export class LoginPage {
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(private page: Page) {
    this.emailInput = page.getByLabel('Email');
    this.passwordInput = page.getByLabel('Password');
    this.submitButton = page.getByRole('button', { name: /sign in/i });
    this.errorMessage = page.getByRole('alert');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
}
```

### Test with assertions

```ts
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from './pages/login.page';

test.describe('Authentication', () => {
  test('successful login redirects to dashboard', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login('alice@example.com', 'password123');

    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByRole('heading', { name: /dashboard/i })).toBeVisible();
  });

  test('invalid credentials show error', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login('alice@example.com', 'wrong');

    await expect(loginPage.errorMessage).toHaveText(/invalid credentials/i);
    await expect(page).toHaveURL('/login');
  });

  test('login form is keyboard accessible', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();

    await page.keyboard.press('Tab');
    await expect(loginPage.emailInput).toBeFocused();

    await page.keyboard.type('alice@example.com');
    await page.keyboard.press('Tab');
    await expect(loginPage.passwordInput).toBeFocused();

    await page.keyboard.type('password123');
    await page.keyboard.press('Enter');

    await expect(page).toHaveURL('/dashboard');
  });
});
```

### Accessibility checks in E2E

```ts
// e2e/a11y.spec.ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

const pages = ['/', '/login', '/dashboard', '/settings'];

for (const path of pages) {
  test(`${path} has no accessibility violations`, async ({ page }) => {
    await page.goto(path);
    const results = await new AxeBuilder({ page }).analyze();
    expect(results.violations).toEqual([]);
  });
}
```