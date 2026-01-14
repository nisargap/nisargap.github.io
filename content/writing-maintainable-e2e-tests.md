+++
title = "Writing E2E Tests That Don't Break"
date = 2026-01-07
description = "Flaky tests are worse than no tests. Here's how to write E2E tests that stay reliable."
[taxonomies]
tags = ["testing", "e2e", "playwright", "best-practices"]
categories = ["Testing"]
+++

E2E tests have a reputation for flakiness. Tests that pass locally fail in CI. Tests that passed yesterday fail today. Teams lose trust and eventually disable the tests entirely.

This doesn't have to happen. Flaky tests are usually a symptom of poor test design, not an inherent property of E2E testing.

<!-- more -->

## The Sources of Flakiness

Before fixing flakiness, understand where it comes from:

1. **Timing issues**: Tests that don't wait for elements properly
2. **Test interdependence**: Tests that depend on state from previous tests
3. **Brittle selectors**: Locators that break when unrelated markup changes
4. **Environment inconsistency**: Tests that assume specific data or configuration
5. **Resource contention**: Tests competing for the same resources

Each source has specific remedies.

## Wait for Reality, Not Time

The most common cause of flakiness is arbitrary waits:

```javascript
// BAD: arbitrary sleep
await page.click('#submit');
await page.waitForTimeout(2000);
await expect(page.locator('#result')).toBeVisible();

// GOOD: wait for the actual condition
await page.click('#submit');
await expect(page.locator('#result')).toBeVisible();
```

Modern testing frameworks like Playwright have auto-waiting built in. Use it:

```javascript
// Playwright automatically waits for elements to be actionable
await page.click('#submit');  // Waits for button to be visible and enabled
await page.fill('#email', 'test@example.com');  // Waits for input to be editable
```

For complex conditions, wait explicitly:

```javascript
// Wait for network requests to complete
await page.waitForResponse(response =>
    response.url().includes('/api/data') && response.status() === 200
);

// Wait for specific DOM state
await page.waitForFunction(() =>
    document.querySelectorAll('.item').length >= 10
);
```

## Use Resilient Selectors

Selectors based on styling or structure break constantly:

```javascript
// BRITTLE: breaks if structure changes
await page.click('div > div > button.btn-primary');

// BRITTLE: breaks if styling changes
await page.click('.submit-button-blue-large');

// ROBUST: uses test-specific attributes
await page.click('[data-testid="submit-button"]');

// ROBUST: uses accessible roles
await page.getByRole('button', { name: 'Submit' });
```

Add `data-testid` attributes to your components:

```jsx
// React example
<button data-testid="submit-order" onClick={handleSubmit}>
    Complete Purchase
</button>
```

This creates a contract between tests and code that's independent of styling and structure.

## Isolate Tests Completely

Tests that share state are tests that flake:

```javascript
// BAD: tests share state
describe('User management', () => {
    test('create user', async () => {
        await createUser('testuser');
    });

    test('delete user', async () => {
        // Fails if 'create user' didn't run first
        await deleteUser('testuser');
    });
});

// GOOD: each test is self-contained
describe('User management', () => {
    test('create user', async () => {
        const username = `user-${Date.now()}`;
        await createUser(username);
        await expect(page.locator(`[data-user="${username}"]`)).toBeVisible();
    });

    test('delete user', async () => {
        const username = `user-${Date.now()}`;
        await createUser(username);  // Setup within the test
        await deleteUser(username);
        await expect(page.locator(`[data-user="${username}"]`)).not.toBeVisible();
    });
});
```

Use unique identifiers for test data to prevent collisions:

```javascript
function uniqueEmail() {
    return `test-${Date.now()}-${Math.random().toString(36).slice(2)}@example.com`;
}
```

## Reset State Between Tests

Use hooks to ensure clean state:

```javascript
// Playwright example
test.beforeEach(async ({ page }) => {
    // Clear cookies and storage
    await page.context().clearCookies();

    // Reset database via API
    await fetch('http://localhost:3000/api/test/reset', { method: 'POST' });

    // Start from known state
    await page.goto('/');
});
```

For more complex scenarios, use database transactions that rollback:

```javascript
test.beforeEach(async () => {
    await db.query('BEGIN');
    await db.query('SAVEPOINT test_start');
});

test.afterEach(async () => {
    await db.query('ROLLBACK TO SAVEPOINT test_start');
    await db.query('ROLLBACK');
});
```

## Handle Animations and Transitions

Animations cause tests to interact with elements before they're ready:

```javascript
// Option 1: Disable animations in tests
await page.addStyleTag({
    content: `
        *, *::before, *::after {
            animation-duration: 0s !important;
            transition-duration: 0s !important;
        }
    `
});

// Option 2: Wait for animations to complete
await page.locator('#modal').waitFor({ state: 'visible' });
await page.locator('#modal').evaluate(el =>
    el.getAnimations().forEach(a => a.finish())
);
```

## Network Reliability

Tests that depend on external services are inherently flaky. Mock them:

```javascript
// Playwright: intercept and mock API calls
await page.route('**/api/users', route => {
    route.fulfill({
        status: 200,
        contentType: 'application/json',
        body: JSON.stringify([
            { id: 1, name: 'Test User' }
        ])
    });
});
```

For internal APIs, use test fixtures:

```javascript
// Seed predictable data before tests
test.beforeAll(async () => {
    await seedTestData({
        users: [{ id: 1, email: 'test@example.com' }],
        products: [{ id: 1, name: 'Widget', price: 10 }]
    });
});
```

## Debugging Flaky Tests

When tests flake, you need visibility:

**Screenshots on failure**:
```javascript
// Playwright does this automatically
test.afterEach(async ({ page }, testInfo) => {
    if (testInfo.status !== 'passed') {
        await page.screenshot({
            path: `screenshots/${testInfo.title}.png`,
            fullPage: true
        });
    }
});
```

**Video recording**:
```javascript
// playwright.config.js
module.exports = {
    use: {
        video: 'retain-on-failure'
    }
};
```

**Trace files**:
```javascript
// playwright.config.js
module.exports = {
    use: {
        trace: 'retain-on-failure'
    }
};
```

Trace files are especially powerful—they capture every action, network request, and console log, letting you replay exactly what happened.

## Retry Strategically

Some flakiness is environmental and unavoidable. Strategic retries help:

```javascript
// playwright.config.js
module.exports = {
    retries: process.env.CI ? 2 : 0,  // Retry in CI, not locally
};
```

But don't use retries to mask bad tests. If a test needs more than 2 retries, fix the underlying issue.

## The Maintenance Contract

E2E tests are code. They need the same care as production code:

- **Review tests in PRs**: Catch brittle patterns before they merge
- **Refactor utilities**: Extract common patterns into helpers
- **Delete obsolete tests**: Dead tests are worse than no tests
- **Monitor flakiness**: Track which tests fail intermittently and fix them

A well-maintained E2E suite is an asset. A neglected one is a liability. The investment you make in test quality pays dividends in deployment confidence.
