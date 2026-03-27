# Prompt: Set Up a Vitest + Playwright + Storybook Testing Framework in a TypeScript Monorepo

Use this prompt as a comprehensive guide (or as an AI prompt) to set up a production-grade testing framework from scratch in a TypeScript monorepo. It is tool-agnostic in its patterns but prescriptive in its architecture.

---

## Context

You are setting up the complete testing infrastructure for a **TypeScript monorepo** that contains:

- Multiple **frontend applications** (React / Next.js)
- Multiple **backend microservices** (Node.js / Fastify / Hono)
- **Shared libraries** consumed by both frontend and backend
- A **component library** (design system)
- A **monorepo task runner** (Nx, Turborepo, or similar)

The testing pyramid consists of three layers:

1. **Vitest** — Unit and integration tests
2. **Playwright** — End-to-end browser tests
3. **Storybook** — Component stories, interaction tests, and visual regression

---

## Part 1: Vitest Setup (Unit & Integration Testing)

### 1.1 Shared Configuration Factory

Create shared Vitest config factory functions that projects import and extend. This prevents config drift across 50+ projects while allowing project-specific overrides.

**Requirements:**

- Create a `configs/vitest/` directory at the workspace root with:
  - `index.ts` — Export factory functions: `defineFrontendConfig()`, `defineBackendUnitConfig()`, `defineIntegrationConfig()`
  - `vitest.setup.ts` — Shared setup file (testing-library cleanup, global env vars, optional secrets bootstrap)

- **`defineFrontendConfig()`** should:
  - Use `@vitejs/plugin-react` for JSX transform
  - Set `environment: 'jsdom'` (or let projects override to `happy-dom`)
  - Enable `globals: true` for cleaner test files
  - Include `**/*.test.{ts,tsx}` files
  - Exclude `node_modules`, `.next`, `dist`
  - Reference the shared setup file
  - Set `passWithNoTests: true` for new projects

- **`defineBackendUnitConfig()`** should:
  - Use `environment: 'node'`
  - Include `**/*.test.ts` and `**/*.spec.ts`
  - **Exclude** `**/*.int.test.ts` (integration tests run separately)
  - Set reasonable timeouts for async operations
  - Optionally configure `env` for common test environment variables

- **`defineIntegrationConfig()`** should:
  - Include **only** `**/*.int.test.ts` files
  - Use `environment: 'node'`
  - Set `fileParallelism: false` (integration tests often share DB state)
  - Set `pool: 'forks'` for process isolation

- Each factory should accept an optional override object that gets deep-merged:

```typescript
export function defineFrontendConfig(overrides?: Partial<UserConfig>): UserConfig {
  return mergeConfig(baseConfig, defineConfig({ ...defaults, ...overrides }));
}
```

### 1.2 Per-Project Configuration

Each project creates a thin `vitest.config.ts` that imports and extends the shared factory:

```typescript
// apps/my-app/vitest.config.ts
import { mergeConfig, defineConfig } from 'vitest/config';
import { defineFrontendConfig } from '../../configs/vitest';

export default mergeConfig(
  defineFrontendConfig(),
  defineConfig({
    test: {
      alias: { '~': path.resolve(__dirname, './src') },
    },
  }),
);
```

For microservices that need both unit and integration tests, create two config files:

```
apps/my-service/
├── vitest.config.unit.ts       # imports defineBackendUnitConfig()
└── vitest.config.integration.ts # imports defineIntegrationConfig()
```

### 1.3 Mocking Strategy

Establish these mocking conventions from day one:

| Layer | Approach | When |
|-------|----------|------|
| Module mocks | `vi.mock('module')` | Replacing internal modules |
| HTTP mocks | MSW (`setupServer` from `msw/node`) | Any test calling HTTP APIs |
| Domain mocks | Shared mock factories in `library/test-utils/` | Reusable across projects (e.g., `mockUser()`, `mockConfig()`) |

Create a shared test utilities library:

```
library/test-utils/
├── src/
│   ├── factories/        # Mock data factories (use faker or similar)
│   ├── msw/              # Shared MSW handlers
│   ├── vitest/           # Custom matchers, helpers
│   └── index.ts
├── package.json          # @org/test-utils
└── vitest.config.ts
```

### 1.4 Coverage Configuration

Set up coverage as an opt-in but standardized pattern:

- Provider: `v8` (faster than `istanbul`)
- Reporters: `text` (terminal), `lcov` (CI tooling), `html` (local browsing)
- Output path convention: `{workspaceRoot}/coverage/{project-name}/`
- Define threshold presets that projects can reference:

```typescript
export const COVERAGE_THRESHOLDS = {
  strict: { lines: 80, functions: 80, branches: 75, statements: 80 },
  relaxed: { lines: 60, functions: 60, branches: 50, statements: 60 },
} as const;
```

### 1.5 Task Runner Integration

Configure the monorepo task runner to:

- **Infer** a `test` (or `test-vite`) target for any project with a `vitest.config.ts`
- **Cache** unit test results (input: source files + test files + shared configs)
- **Not cache** integration tests (they depend on external services)
- **Depend on** library builds (`^build`) so tests always run against current code
- Define **named inputs** for cache invalidation that reference actual config file paths

### 1.6 File Naming Conventions

Enforce these conventions project-wide:

| Pattern | Runner | Purpose |
|---------|--------|---------|
| `*.test.ts` / `*.test.tsx` | Vitest | Unit tests, colocated with source |
| `*.int.test.ts` | Vitest (integration config) | Integration tests requiring external services |
| `*.spec.ts` | Playwright | E2E tests in `tests/playwright/` |
| `*.stories.tsx` | Storybook | Component stories |
| `*.acceptance.stories.tsx` | Storybook | Acceptance criteria stories with play functions |

---

## Part 2: Playwright Setup (E2E Testing)

### 2.1 Shared Configuration Factory

Mirror the Vitest pattern with composable Playwright config factories:

```
configs/playwright/
├── configs.ts           # Factory functions
├── ci.ts                # export default makeCIConfig({})
├── local.ts             # export default makeLocalConfig({})
└── testing-strategy.json # Optional: branch-based tag selection
```

**`makeBasePlaywrightConfig(config)`** should:

- Set `testDir` to `{cwd}/tests/playwright`
- Set `testMatch` to `**/*.spec.ts`
- Enable `failOnFlakyTests: true`
- Configure reporters: `list` (with `printSteps: true`) + `html` (to `test-results/`)
- Enable `trace: 'on'` for all tests (invaluable for debugging)
- Define three browser projects: chromium, firefox, webkit
- Auto-detect and wire `global-setup.ts` / `global-teardown.ts` if they exist in the test directory
- Accept and spread an override config object

**`makeCIPlaywrightConfig(config)`** should extend base with:

- `forbidOnly: true`
- `retries: 0` (or a small number like 1 for known-flaky environments)
- Dynamic worker count based on environment variables (`PARALLEL=true`, `WORKER_RATIO`)
- `fullyParallel: true` when workers > 1

**`makeLocalConfig(config)`** should extend base with:

- Longer timeout (120s)
- Single worker
- Headed mode (`headless: false`)
- `slowMo: 300` for visual debugging

### 2.2 Layered Test Architecture

Enforce a strict three-layer architecture for all E2E tests. This is the most important design decision for maintainability at scale.

```
tests/playwright/
├── fixtures/          # test.extend() fixture definitions
├── base/              # Manager class + test data managers
├── models/            # Page Object Models (one class per screen/component)
├── pages/             # Screen-level helper functions
├── steps/             # Multi-screen flow orchestration
├── journeys/          # End-to-end journey specs (.spec.ts)
├── screens/           # Single-screen focused specs (.spec.ts)
├── helpers/           # Shared utilities
├── global-setup.ts    # Optional: MSW, auth tokens, DB seeding
├── global-teardown.ts # Optional: cleanup
└── README.md          # Architecture documentation
```

#### Layer 1: Models (Page Objects)

Each screen gets a class that encapsulates locators, actions, and assertions:

```typescript
import { expect } from '@playwright/test';
import { step } from '@org/playwright-utils';

export class LoginPage {
  protected readonly rootLocator = this.page.getByTestId('login-screen');
  protected readonly usernameInput = this.page.getByTestId('username-input');
  protected readonly passwordInput = this.page.getByTestId('password-input');
  protected readonly submitButton = this.page.getByTestId('login-submit');
  protected readonly errorMessage = this.page.getByTestId('login-error');

  constructor(protected readonly page: Page) {}

  @step('Fill login form with {{username}}')
  async fillCredentials(username: string, password: string) {
    await this.usernameInput.fill(username);
    await this.passwordInput.fill(password);
  }

  @step('Submit login form')
  async submit() {
    await this.submitButton.click();
  }

  @step('Verify login error visible')
  async verifyErrorVisible(message?: string) {
    await expect(this.errorMessage).toBeVisible();
    if (message) {
      await expect(this.errorMessage).toContainText(message);
    }
  }

  @step('Verify login screen loaded')
  async verifyLoaded() {
    await expect(this.rootLocator).toBeVisible();
  }
}
```

**Rules for models:**
- `protected readonly rootLocator` on every model
- `@step('...')` decorator on all public methods
- `verify*` methods with `expect()` assertions
- Locators never leak outside the class
- No test logic — only UI interaction and assertion primitives

#### Layer 2: Steps / Pages (Flow Orchestration)

Compose models into reusable multi-screen flows:

```typescript
export async function completeLoginFlow(page: Page, credentials: Credentials) {
  await test.step('Complete login flow', async () => {
    const loginPage = new LoginPage(page);
    await loginPage.verifyLoaded();
    await loginPage.fillCredentials(credentials.username, credentials.password);
    await loginPage.submit();
  });
}
```

**When to extract to steps:**
- Same sequence of 5+ calls repeated across 3+ specs
- Multi-screen flow that constitutes a logical business operation

#### Layer 3: Specs (Test Scenarios)

Specs read like user stories with numbered steps:

```typescript
import { test } from '../fixtures/app.fixtures';

test.describe('Login - Error Handling', { tag: ['@auth', '@regression'] }, () => {
  test.beforeEach(async ({ manager }) => {
    // Precondition: Navigate to login screen
    await manager.navigation.goToLogin();
  });

  test('TC01 - Shows error for invalid credentials', async ({ manager }) => {
    // Step 1: Enter invalid credentials
    await manager.login.fillCredentials('invalid@test.com', 'wrongpass');

    // Step 2: Submit form
    await manager.login.submit();

    // Step 3: Verify error displayed
    await manager.login.verifyErrorVisible('Invalid username or password');
  });
});
```

**Spec rules:**
- No raw `page.getBy*` selectors
- `// Precondition:` comments in `beforeEach`
- `// Step N:` numbered comments in test body
- Test names prefixed `TC01 -`, `TC02 -`
- Tags: `@functional`, `@e2e`, `@regression`, `@TICKET-XXX`
- Every test ends with a `verify*` / `expect*` call

### 2.3 Fixture Pattern

Define typed fixtures that provide a Manager (composition root) and test data:

```typescript
import { test as base } from '@playwright/test';

type AppFixtures = {
  data: TestDataManager;
  manager: AppManager;
};

export const test = base.extend<AppFixtures>({
  data: async ({}, use) => {
    const data = new TestDataManager();
    await use(data);
    await data.teardown();
  },
  manager: async ({ page, data }, use) => {
    await use(new AppManager(page, data));
  },
});

export { expect } from '@playwright/test';
```

The **Manager** is a composition root:

```typescript
export class AppManager {
  readonly login: LoginPage;
  readonly dashboard: DashboardPage;
  readonly navigation: NavigationSteps;

  constructor(public readonly page: Page, public readonly data: TestDataManager) {
    this.login = new LoginPage(page);
    this.dashboard = new DashboardPage(page);
    this.navigation = new NavigationSteps(page);
  }
}
```

### 2.4 `@step` Decorator

Create a shared Playwright utilities library with a `@step` decorator:

```
library/playwright-utils/
├── src/
│   ├── decorators/
│   │   └── step.decorator.ts   # Wraps methods in test.step()
│   ├── BasePage.ts             # Abstract base with goto/waitForURL
│   ├── helpers/                # Accessibility, email, etc.
│   └── index.ts
└── package.json                # @org/playwright-utils
```

The decorator should:
- Wrap the method body in `test.step(stepName, fn, { box: true })`
- Support `{{paramName}}` interpolation in step names
- Auto-stringify object parameters to JSON
- Fall back to `ClassName.methodName` if no message provided

### 2.5 Global Setup Pattern

Use conditional global setup that auto-detects what's needed:

```typescript
// tests/playwright/global-setup.ts
export default async function globalSetup() {
  if (process.env.MOCK_HTTP === 'true') {
    return setupMSW();  // Returns teardown function
  }
}
```

### 2.6 CI Worker Strategy

Implement a dynamic worker count:

```typescript
function getCIWorkerCount(): number {
  if (process.env.PARALLEL === 'true') {
    const ratio = parseFloat(process.env.WORKER_RATIO || '1');
    return Math.max(1, Math.ceil(os.cpus().length * ratio));
  }
  return 1;
}
```

Apps with shared mutable state (MSW, DB) should run with `PARALLEL=false` until state isolation is solved.

### 2.7 Per-Browser Isolation

When running cross-browser in parallel, each browser worker hits the same backend. Solve this by:

- Assigning unique pre-provisioned test users per browser
- Using browser-specific API keys or tokens
- Isolating state per `browserName` in fixtures

```typescript
const usersPerBrowser: Record<string, string> = {
  chromium: 'test-user-chrome',
  firefox: 'test-user-firefox',
  webkit: 'test-user-webkit',
};
```

---

## Part 3: Storybook Setup (Component Testing & Visual Regression)

### 3.1 Shared Configuration

Create a shared Storybook config for all Next.js/React apps:

```
configs/storybook/
├── main.ts           # Framework, addons, Vite config overrides
├── preview.ts        # Global decorators, parameters, viewport presets
├── test-runner.ts    # preVisit/postVisit hooks for test runner
└── ci-config.ts      # Worker count helper for CI
```

**`main.ts`** should:

- Use `@storybook/nextjs-vite` (or `@storybook/react-vite` for non-Next apps)
- Configure story globs: `'../src/**/*.@(mdx|stories.@(js|jsx|ts|tsx))'`
- Add core addons: `@storybook/addon-docs`, `@chromatic-com/storybook`
- Add optional addons per project type: `@storybook/addon-a11y`, `@storybook/addon-themes`, `@storybook/addon-designs`
- Override Vite config to:
  - Add `vite-plugin-node-polyfills` (prevents Node API errors in browser builds)
  - Alias server-only modules to mock stubs
  - Suppress `'use client'` directive warnings
  - Disable sourcemaps in production builds

**`preview.ts`** should:

- Set default layout to `'fullscreen'` (or `'centered'` for component library)
- Configure viewport presets (mobile, tablet, desktop)
- Export parameters and decorators for app-level composition

### 3.2 Per-App Configuration

Each app creates a thin `.storybook/` directory:

```
apps/my-app/.storybook/
├── main.ts          # Extends shared, adds app-specific overrides
├── preview.tsx      # Wraps stories in app-specific providers
└── mocks/           # Module stubs for server-only code
    ├── analytics.ts
    └── server-lib.ts
```

**`preview.tsx`** should wrap stories in the app's provider tree:

```tsx
const preview: Preview = {
  decorators: [
    (Story) => (
      <ThemeProvider theme="default">
        <QueryClientProvider client={queryClient}>
          <Story />
        </QueryClientProvider>
      </ThemeProvider>
    ),
  ],
};
export default preview;
```

### 3.3 Story Patterns (CSF3)

Enforce Component Story Format 3 throughout:

```typescript
import type { Meta, StoryObj } from '@storybook/react';

const meta = {
  title: 'Components/Button',
  component: Button,
  tags: ['autodocs'],
} satisfies Meta<typeof Button>;

export default meta;
type Story = StoryObj<typeof meta>;

export const Primary: Story = {
  args: { variant: 'primary', children: 'Click me' },
};

export const WithInteraction: Story = {
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);
    await userEvent.click(canvas.getByRole('button'));
    await expect(canvas.getByText('Clicked!')).toBeVisible();
  },
};
```

### 3.4 Acceptance Test Stories

Define a pattern for encoding acceptance criteria as stories:

```typescript
// ComponentName.acceptance.stories.tsx
const meta = {
  title: 'Acceptance/TICKET-123/Feature Name',
  component: FeatureScreen,
  tags: ['@feature', '@TICKET-123'],
  decorators: [/* required providers */],
} satisfies Meta<typeof FeatureScreen>;

export default meta;

export const TC01_HappyPath: Story = {
  name: 'TC01 - User completes action successfully',
  args: { /* scenario-specific props */ },
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);
    // Interact and assert
    await userEvent.click(canvas.getByRole('button', { name: 'Submit' }));
    await expect(canvas.getByText('Success')).toBeVisible();
  },
};
```

This pattern:
- Links stories to tickets via `title` and `tags`
- Uses test case numbering (`TC01`, `TC02`) for traceability
- Tests real component behavior via `play` functions
- Runs in both Storybook UI and headless via test-runner

### 3.5 Storybook Test Runner

Configure `@storybook/test-runner` for CI:

```typescript
// configs/storybook/test-runner.ts
import type { TestRunnerConfig } from '@storybook/test-runner';

const config: TestRunnerConfig = {
  async preVisit(page) {
    await page.waitForLoadState('domcontentloaded');
    await page.waitForLoadState('networkidle');
  },
  async postVisit(page) {
    await page.focus('body');  // Reset focus state between stories
  },
};

export default config;
```

**CI execution pattern:**
1. Build Storybook to static files (`storybook build`)
2. Serve static files on a predictable port
3. Run test-runner against the served URL
4. Cache the result (inputs: stories + .storybook config)

### 3.6 Visual Regression (Chromatic)

For each project that needs visual regression:

```json
// chromatic.config.json
{
  "projectId": "your-project-id",
  "buildScriptName": "storybook:build",
  "outputDir": "storybook-static",
  "onlyChanged": true,
  "externals": ["public/**"]
}
```

For the component library, configure multi-theme snapshots:

```typescript
// preview.tsx
parameters: {
  chromatic: {
    modes: {
      'light-theme': { theme: 'light' },
      'dark-theme': { theme: 'dark' },
    },
  },
},
```

---

## Part 4: Task Runner Integration (Nx Example)

### 4.1 Target Definitions

Define these targets in the task runner config:

| Target | Tool | Cached | Dependencies |
|--------|------|--------|-------------|
| `test` | Vitest | Yes | `^build` (library builds) |
| `test:integration` | Vitest (integration config) | No | `^build` |
| `test:e2e` | Playwright | No | `^build`, shared playwright lib build, browser install, app start |
| `storybook` | Storybook dev | No | — |
| `storybook:build` | Storybook build | Yes | — |
| `storybook:test` | Test runner | Yes | browser install, `storybook:build` |

### 4.2 Named Inputs for Cache Invalidation

```json
{
  "namedInputs": {
    "testing": [
      "{projectRoot}/src/**/*",
      "{projectRoot}/**/*.test.*",
      "{projectRoot}/**/*.spec.*",
      "{projectRoot}/tests/**/*",
      "{workspaceRoot}/configs/vitest/**/*",
      "{workspaceRoot}/configs/playwright/**/*"
    ],
    "storybook": [
      "{projectRoot}/**/*.stories.*",
      "{projectRoot}/.storybook/**/*",
      "{workspaceRoot}/configs/storybook/**/*"
    ]
  }
}
```

**Verify that every referenced path actually exists.** Stale references cause incorrect cache behavior.

### 4.3 Plugin-Based Target Inference

Register framework plugins so projects with config files automatically get targets:

```json
{
  "plugins": [
    { "plugin": "@nx/vite/plugin", "options": { "testTargetName": "test" } },
    { "plugin": "@nx/storybook/plugin", "options": { "buildStorybookTargetName": "storybook:build", "testStorybookTargetName": "storybook:test" } },
    { "plugin": "@nx/playwright/plugin", "options": { "targetName": "test:e2e" } }
  ]
}
```

---

## Part 5: CI Pipeline

### 5.1 PR Workflow

```yaml
jobs:
  unit-tests:
    steps:
      - run: pnpm nx affected -t test --base=origin/main

  e2e-tests:
    strategy:
      matrix:
        project: [from affected projects with test:e2e target]
    steps:
      - run: pnpm exec playwright install --with-deps
      - run: pnpm nx run ${{ matrix.project }}:test:e2e
    env:
      PARALLEL: true
      WORKER_RATIO: '0.5'   # For heavy suites
    on-failure:
      - upload-artifact: **/test-results

  storybook-tests:
    steps:
      - run: pnpm exec playwright install --with-deps  # test-runner needs Playwright
      - run: pnpm nx affected -t storybook:test

  integration-tests:
    services:
      postgres: { image: postgres:16 }
    steps:
      - run: pnpm nx affected -t test:integration
    env:
      DATABASE_URL: postgresql://...

  visual-regression:
    needs: storybook-tests
    steps:
      - run: npx chromatic --project-token=${{ secrets.CHROMATIC_TOKEN }}
```

### 5.2 Cache Persistence

Save and restore the task runner cache across CI runs:

```yaml
- uses: actions/cache@v4
  with:
    path: .nx
    key: nx-${{ github.ref }}-${{ github.sha }}
    restore-keys: |
      nx-${{ github.ref }}-
      nx-refs/heads/main-
```

---

## Part 6: Shared Libraries

Create these shared packages:

### `@org/test-utils` (for Vitest)

```
library/test-utils/src/
├── factories/          # Mock data factories
├── msw/                # Shared MSW handlers and setup
├── vitest/             # Custom matchers, helpers
├── reporters/          # Custom Vitest reporters (e.g., Xray integration)
└── index.ts
```

### `@org/playwright-utils` (for Playwright)

```
library/playwright-utils/src/
├── decorators/
│   └── step.decorator.ts    # @step method decorator
├── BasePage.ts              # Abstract base page with navigation helpers
├── helpers/
│   ├── accessibility.ts     # Axe integration helpers
│   └── email.ts             # Email verification (MailSlurp, etc.)
├── reporters/
│   └── xray.reporter.ts     # Test management integration
└── index.ts
```

Both libraries must be **built before tests run** (wire as dependencies in task runner config).

---

## Part 7: Documentation

Each app with E2E tests must have a `tests/playwright/TESTING.md` (or `README.md`) covering:

1. Architecture diagram (the three layers)
2. Directory structure with descriptions
3. Fixture usage examples
4. How to add a new model, step, spec
5. Anti-patterns list
6. Environment levels (local, CI, staging)
7. How mocking works (MSW, route interception)
8. Checklist for new tests

---

## Implementation Order

1. **Shared configs** — Vitest factories, Playwright factories, Storybook shared config
2. **Shared libraries** — `@org/test-utils`, `@org/playwright-utils`
3. **Task runner targets** — Plugin registration, named inputs, target defaults
4. **Pilot app** — Set up one app with all three frameworks as the reference implementation
5. **CI pipeline** — Unit → E2E → Storybook → Visual regression
6. **Migration** — Roll out to remaining apps, one at a time
7. **Documentation** — TESTING.md per app, root-level testing guide
8. **Coverage & reporting** — Coverage thresholds, Xray integration, Chromatic

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why | Instead |
|-------------|-----|---------|
| Per-project configs with no shared base | Config drift, duplicated boilerplate | Factory functions in `configs/` |
| Raw `page.getBy*` in spec files | Specs become unreadable and brittle | Page Object Model classes |
| Copy-pasted `beforeEach` setup | Maintenance nightmare | Fixtures + Manager pattern |
| Caching E2E tests | External state makes results non-deterministic | `cache: false` for E2E |
| `retries: 3` to hide flaky tests | Masks real issues | `failOnFlakyTests: true`, fix root causes |
| Empty vitest configs | Confusing, serves no purpose | Only create when tests exist |
| Mixing `.spec.ts` and `.test.ts` in same directory | Ambiguous which runner executes them | Enforce location-based convention |
| Assertions in Page Objects | Couples UI interaction with test logic | Separate `verify*` methods using `expect()` |
| Hardcoded test users across parallel browsers | State collisions when browsers share backend | Per-browser user isolation |
| Global `any` in test utilities | Defeats TypeScript's purpose | Proper generics and type guards |
| Testing implementation details | Tests break on refactors | Test behavior and user-visible outcomes |
| Storybook without `play` functions | Stories are only visual, not tested | Add interaction tests for key behaviors |
| Skipping stories for complex components | Loses the most valuable coverage | Mock at the boundary, test the component |

---

## Validation Checklist

Before considering the setup complete, verify:

- [ ] A new project can get unit tests running by creating one `vitest.config.ts` + one `.test.ts` file
- [ ] E2E tests run cross-browser (chromium, firefox, webkit) locally and in CI
- [ ] Storybook builds and test-runner passes for all configured projects
- [ ] CI runs only affected tests on PRs
- [ ] Task runner cache correctly invalidates when shared configs change
- [ ] Test traces and HTML reports are generated for Playwright
- [ ] Chromatic (or equivalent) catches visual regressions
- [ ] Named inputs reference paths that actually exist
- [ ] All shared libraries build before dependent tests run
- [ ] Documentation exists for at least the pilot app
- [ ] `failOnFlakyTests` is enabled for Playwright
- [ ] Integration tests run with real services (DB) in CI

---

*This prompt captures the architectural patterns for a production-grade testing setup. Adapt the specific tool versions, directory names, and CI provider to your stack.*
