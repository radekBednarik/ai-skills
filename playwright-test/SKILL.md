---
name: playwright-test-best-practices
description: >
  Guidelines for writing high-quality Playwright TypeScript tests. Load this
  skill when generating, reviewing, or refactoring test code to ensure it
  follows official best practices, uses Page Object Model for pages, components
  and modals, respects DRY and KISS principles, and avoids deprecated APIs.
allowed-tools: Bash(pnpm:*), Bash(tsc:*), Bash(eslint:*), Bash(prettier:*)
---

# Playwright Test Best Practices

This skill defines the rules and conventions that **all generated and reviewed Playwright TypeScript test code must follow**. Apply every section below without exception.

---

## 1. Testing Philosophy

### Test user-visible behavior

Tests must verify what the end user sees and interacts with — not internal implementation details like function names, CSS classes, array shapes, or component internals.

```typescript
// Good — tests what the user sees
await expect(page.getByRole("heading", { name: "Welcome back" })).toBeVisible();

// Bad — tests implementation detail
await expect(page.locator(".welcome-heading.text-xl")).toBeVisible();
```

### Make tests fully isolated

Every test must run independently with its own browser context, local storage, session storage, cookies and data. Tests must never share mutable state or depend on execution order.

Use `test.beforeEach` for shared setup (navigation, login). Use Playwright's [recommended approaches](https://playwright.dev/docs/auth#introduction) to reuse authenticated state across tests without repeating login steps in every file.

```typescript
import { test } from "@playwright/test";

test.beforeEach(async ({ page }) => {
  await page.goto("https://example.com/login");
  await page.getByLabel("Email").fill("user@example.com");
  await page.getByLabel("Password").fill("secret");
  await page.getByRole("button", { name: "Sign in" }).click();
});
```

### Avoid testing third-party dependencies

Never write tests that rely on external services, third-party APIs, or content you do not control. Mock external network calls using `page.route()`.

```typescript
await page.route("**/api/external-service/**", (route) =>
  route.fulfill({ status: 200, body: JSON.stringify({ ok: true }) }),
);
```

### Control test data

Test against a staging environment with deterministic, controlled data. For visual regression tests, pin OS and browser versions.

---

## 2. Locator Strategy

Always prefer locators that are resilient to DOM changes and reflect how users experience the UI. Apply this priority order strictly:

| Priority | Locator                                        | When to use                             |
| -------- | ---------------------------------------------- | --------------------------------------- |
| 1        | `page.getByRole('button', { name: 'Submit' })` | Interactive elements with ARIA roles    |
| 2        | `page.getByLabel('Email')`                     | Form inputs associated with a `<label>` |
| 3        | `page.getByPlaceholder('Search...')`           | Inputs with placeholder text            |
| 4        | `page.getByText('Welcome back')`               | Static text content                     |
| 5        | `page.getByTestId('submit-btn')`               | Elements with `data-testid` attributes  |
| 6        | `page.locator('css=...')`                      | **Last resort only** — fragile, avoid   |

**Never use:**

- XPath selectors
- CSS class selectors based on visual styling (`.btn-primary`, `.text-xl`)
- Positional selectors (`.nth(0)`, `.first()`) unless no alternative exists
- `page.$()`, `page.$$()` — these return `ElementHandle` which is deprecated

### Chaining and filtering

Scope locators to a container to disambiguate, rather than relying on index.

```typescript
// Good — scoped and semantic
const cartItem = page.getByRole("listitem").filter({ hasText: "Product 2" });
await cartItem.getByRole("button", { name: "Remove" }).click();

// Bad — positional, fragile
await page.getByRole("button", { name: "Remove" }).nth(1).click();
```

---

## 3. Assertions

### Use web-first assertions exclusively

Web-first assertions (`expect(locator).toBeVisible()` etc.) auto-retry until the condition is met or the timeout expires. Always `await` the entire `expect(...)` expression.

```typescript
// Good — web-first, auto-retries
await expect(page.getByRole("alert")).toHaveText("Saved successfully");
await expect(page).toHaveURL("/dashboard");
await expect(page.getByRole("button", { name: "Submit" })).toBeEnabled();
await expect(page.getByRole("textbox", { name: "Name" })).toHaveValue("Alice");

// Bad — manual assertion, no retry, races with async rendering
expect(await page.getByRole("alert").isVisible()).toBe(true);
```

### Never use `waitForTimeout`

`page.waitForTimeout()` is a hard sleep — it makes tests slow and flaky. Replace every occurrence with an explicit web-first assertion that waits for the expected state.

```typescript
// Bad
await page.waitForTimeout(2000);
await expect(page.getByRole("alert")).toBeVisible();

// Good
await expect(page.getByRole("alert")).toBeVisible(); // waits automatically
```

### Soft assertions

Use `expect.soft()` when you want to check multiple conditions in one test run without stopping at the first failure. The test is still marked failed if any soft assertion fails.

```typescript
await expect.soft(page.getByTestId("order-status")).toHaveText("Confirmed");
await expect.soft(page.getByTestId("payment-status")).toHaveText("Paid");
// test continues even if one fails, both failures are reported
```

---

## 4. Page Object Model

All test code that interacts with the UI **must** be structured using Page Object Model (POM) classes. Raw Playwright calls (`page.click()`, `page.fill()`) must not appear directly in test files — they belong inside POM classes.

### When to create each type

| Concept                      | Class suffix | Example                                   | Location          |
| ---------------------------- | ------------ | ----------------------------------------- | ----------------- |
| Full navigable page          | `Page`       | `LoginPage`, `DashboardPage`              | `src/pages/`      |
| Reusable page section/widget | `Component`  | `NavbarComponent`, `ProductCardComponent` | `src/components/` |
| Modal or dialog overlay      | `Modal`      | `ConfirmDeleteModal`, `AddAddressModal`   | `src/modals/`     |

### POM rules

- Constructor accepts `Page` from `@playwright/test` as its only parameter.
- All locators are declared as `readonly` properties and initialised in the constructor.
- Methods represent meaningful user actions (`login()`, `addToCart()`, `confirmDeletion()`).
- **No assertions inside POM classes.** Assertions belong exclusively in test files. POMs expose locators as `readonly` so tests can assert against them directly.
- No raw `page.click(selector)` string-overload calls — always use `this.someLocator.click()`.
- Shared methods and properties are defined in an abstract `BasePage` class.
  Specific page POM classes will inherit the `BasePage`.
- Keep the rule, that there should be only one `default` class per module.

### Page class example

```typescript
// src/pages/LoginPage.ts
import type { Page, Locator } from "@playwright/test";

export default class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel("Email");
    this.passwordInput = page.getByLabel("Password");
    this.submitButton = page.getByRole("button", { name: "Sign in" });
    this.errorMessage = page.getByRole("alert");
  }

  async goto(): Promise<void> {
    await this.page.goto("/login");
  }

  async login(email: string, password: string): Promise<void> {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
}
```

### Component class example

```typescript
// src/components/NavbarComponent.ts
import type { Page, Locator } from "@playwright/test";

export default class NavbarComponent {
  readonly page: Page;
  readonly self: Locator;
  readonly accountMenu: Locator;
  readonly cartIcon: Locator;

  constructor(page: Page) {
    this.page = page;
    this.self = this.page.getByRole("navigation");
    this.accountMenu = this.page.getByRole("button", { name: "My account" });
    this.cartIcon = this.page.getByRole("link", { name: "Cart" });
  }

  async openAccountMenu(): Promise<void> {
    await this.accountMenu.click();
  }

  async goToCart(): Promise<void> {
    await this.cartIcon.click();
  }
}
```

### Component integration to Page Objects

You should compose other POM classes as properties to represent nested components.

```typescript
import NavbarComponent from "../components/NavbarComponent";

export default class LoginPage {
  public readonly navbar: NavbarComponent;

  constructor(page: Page) {
    this.navbar = new NavbarComponent(page);
  }
}
```

### Modal class example

```typescript
// src/modals/ConfirmDeleteModal.ts
import type { Page, Locator } from "@playwright/test";

export class ConfirmDeleteModal {
  readonly page: Page;
  readonly heading: Locator;
  readonly confirmButton: Locator;
  readonly cancelButton: Locator;

  constructor(page: Page) {
    this.page = page;
    this.heading = page.getByRole("dialog").getByRole("heading");
    this.confirmButton = page
      .getByRole("dialog")
      .getByRole("button", { name: "Delete" });
    this.cancelButton = page
      .getByRole("dialog")
      .getByRole("button", { name: "Cancel" });
  }

  async confirm(): Promise<void> {
    await this.confirmButton.click();
  }

  async cancel(): Promise<void> {
    await this.cancelButton.click();
  }
}
```

---

## 5. DRY & KISS Principles

### Use fixtures to inject POM instances

Playwright fixtures are the idiomatic way to share POM instances across tests without repeating construction code. Define custom fixtures in a shared file and import `test` from it instead of `@playwright/test`.

```typescript
// src/fixtures.ts
import { test as base } from "@playwright/test";
import { LoginPage } from "./pages/LoginPage";
import { NavbarComponent } from "./components/NavbarComponent";

type Fixtures = {
  loginPage: LoginPage;
  navbar: NavbarComponent;
};

export const test = base.extend<Fixtures>({
  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page));
  },
  navbar: async ({ page }, use) => {
    await use(new NavbarComponent(page));
  },
});

export { expect } from "@playwright/test";
```

```typescript
// src/tests/login.spec.ts
import { test, expect } from "../fixtures";

test("shows error for wrong password", async ({ loginPage }) => {
  await loginPage.goto();
  await loginPage.login("user@example.com", "wrong");
  await expect(loginPage.errorMessage).toHaveText("Invalid credentials");
});
```

### One concern per test

Each test verifies exactly one behaviour. Split multi-scenario tests into separate `test()` blocks.

```typescript
// Bad — two unrelated assertions in one test
test("login page", async ({ loginPage }) => {
  await loginPage.goto();
  await expect(loginPage.submitButton).toBeDisabled();
  await loginPage.login("user@example.com", "wrong");
  await expect(loginPage.errorMessage).toBeVisible();
});

// Good — two focused, independent tests
test("submit is disabled when fields are empty", async ({ loginPage }) => {
  await loginPage.goto();
  await expect(loginPage.submitButton).toBeDisabled();
});

test("shows error for invalid credentials", async ({ loginPage }) => {
  await loginPage.goto();
  await loginPage.login("user@example.com", "wrong");
  await expect(loginPage.errorMessage).toHaveText("Invalid credentials");
});
```

### No duplicate setup code

If multiple tests in a file share the same setup step, move it to `test.beforeEach`. If setup is shared across files, use a fixture or a Playwright setup project.

---

## 6. Deprecated API — Do Not Use

The following Playwright APIs are deprecated or discouraged. **Never generate or use them.**

| Deprecated                                               | Replacement                                           |
| -------------------------------------------------------- | ----------------------------------------------------- |
| `page.waitForSelector(selector)`                         | `await expect(page.locator(selector)).toBeVisible()`  |
| `page.$(selector)`                                       | `page.locator(selector)`                              |
| `page.$$(selector)`                                      | `page.locator(selector)`                              |
| `page.$eval(selector, fn)`                               | `page.locator(selector).evaluate(fn)`                 |
| `page.$$eval(selector, fn)`                              | `page.locator(selector).evaluateAll(fn)`              |
| `page.click(selector)` _(string overload on Page)_       | `page.locator(selector).click()` or a POM method      |
| `page.fill(selector, value)` _(string overload on Page)_ | `page.locator(selector).fill(value)` or a POM method  |
| `page.type(selector, text)`                              | `page.locator(selector).pressSequentially(text)`      |
| `elementHandle.*` (any ElementHandle method)             | Equivalent `Locator` API method                       |
| `page.waitForTimeout(ms)`                                | Web-first assertion that waits for the expected state |

---

## 7. Test Organisation

### Use `test.describe` to group related tests

```typescript
import { test, expect } from "../fixtures";

test.describe("Login flow", () => {
  test("valid credentials redirect to dashboard", async ({ loginPage }) => {
    /* ... */
  });
  test("invalid credentials show error", async ({ loginPage }) => {
    /* ... */
  });
  test("empty form disables submit button", async ({ loginPage }) => {
    /* ... */
  });
});
```

### Use `test.beforeEach` / `test.afterEach` for shared setup

```typescript
test.beforeEach(async ({ loginPage }) => {
  await loginPage.goto();
});
```

### Never commit `test.only`

`forbidOnly` is enforced in CI. Use `--grep` to run a single test during development:

```bash
pnpm exec playwright test --grep "shows error for invalid credentials"
```

### Parallelism

Tests across files run in parallel by default. To run tests within a single file in parallel:

```typescript
test.describe.configure({ mode: "parallel" });
```

Be careful, whether the test user can be shared across tests (if fully parallel mode is on), or across test files, since they may be collisions, if tests are modifying backend state.

If this situation occurs, structure test files in such a way, that each test file uses distinct test client and full parallel mode within these test files is turned off.

```typescript
test.describe.configure({ mode: "serial" });
```

### Test users

Test users are to be specified as Typescript objects in a shared file (e.g. `src/test-users.ts`) and imported where needed. Never hardcode test user credentials directly in test files.

```typescript
import { testUsers } from "../test-users";
```

---

## 8. TypeScript Rules

All generated code must comply with the project's strict TypeScript configuration.

- Use `import type` for imports that are only used as types:
  ```typescript
  import { test, expect } from "@playwright/test";
  import type { Page, Locator } from "@playwright/test";
  ```
- Every Playwright action must be `await`ed.
- All test body functions must be `async`.
- Remove all unused variables and parameters (`noUnusedLocals`, `noUnusedParameters` are enabled).
- All code paths in a function must return explicitly (`noImplicitReturns`).
- Never assign `undefined` to optional properties explicitly (`exactOptionalPropertyTypes`).
- Return type annotations are recommended on POM methods for clarity.
- Use path aliasing for imports if configured (e.g. `@pages/LoginPage` instead of `../pages/LoginPage`).

---

## 9. Verification Checklist

After generating or modifying test code, always run:

```bash
# Type-check
pnpm exec tsc --noEmit

# Lint
pnpm exec eslint src/

# Run the affected test(s)
pnpm exec playwright test src/tests/my-feature.spec.ts --project=chromium
```

All three must pass with zero errors before the code is considered complete.
