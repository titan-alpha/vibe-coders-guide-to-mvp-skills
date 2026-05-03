# 16 · E2E Testing

Goal: drive the MVP end-to-end with a real browser, page by page, and ship nothing without three passes:

- **Pass A — Flow tests.** The user can complete the journey (signup → core feature → sign-out).
- **Pass B — Per-page functional crawl.** Every route loads, has no console errors, no broken links, all interactive elements are reachable.
- **Pass C — Per-page visual audit.** Screenshot every route on every breakpoint and grade each against a structured rubric.

All three run against `localhost:3000` first; Pass A repeats against the production URL after deploy (sub-skill 13).

## DIALOGUE — confirm with the user

Browser automation will open visible windows and exercise every flow, including auth and any AI features. Confirm before kicking off.

> *"I'd like to run end-to-end tests with a real browser in three passes: flow tests, a per-page functional crawl, and a per-page visual audit. The first two are automated and fast (~2 min); the visual audit takes me 5–10 minutes because I'll look at every screenshot and grade it against a rubric. OK to start?"*

If the user has data isolation concerns, ask whether to test against staging or production. If they don't have a staging env, point at production but use a clearly-marked test account (e.g., `e2e+<timestamp>@<domain>`).

## AUTONOMOUS — set up Playwright

```bash
npm install --save-dev @playwright/test playwright
npx playwright install chromium
```

`playwright.config.ts`:

```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: true,
    timeout: 120_000,
  },
  use: {
    baseURL: process.env.E2E_BASE_URL ?? 'http://localhost:3000',
    screenshot: 'only-on-failure',
    trace: 'retain-on-failure',
  },
  projects: [
    { name: 'mobile-sm', use: { ...devices['iPhone SE'] } },                                                      // 375
    { name: 'mobile-lg', use: { ...devices['iPhone 14'] } },                                                      // 390
    { name: 'tablet',    use: { ...devices['iPad Mini'] } },                                                      // 768
    { name: 'desktop',   use: { ...devices['Desktop Chrome'], viewport: { width: 1280, height: 800 } } },
    { name: 'wide',      use: { ...devices['Desktop Chrome'], viewport: { width: 1920, height: 1080 } } },
  ],
});
```

### Build the route manifest

Both Pass B and Pass C consume the same list. Create `e2e/routes.ts`:

```ts
// e2e/routes.ts — KEEP IN SYNC WITH app/ AS PAGES ARE ADDED.
export const ROUTES = [
  { path: '/',          name: 'home',      auth: false },
  { path: '/login',     name: 'login',     auth: false },
  { path: '/about',     name: 'about',     auth: false },
  { path: '/faq',       name: 'faq',       auth: false },
  { path: '/privacy',   name: 'privacy',   auth: false },
  { path: '/terms',     name: 'terms',     auth: false },
  { path: '/dashboard', name: 'dashboard', auth: true  },
  // ...add every route in app/ that returns a page (not API routes).
];
```

For authed routes, set up `globalSetup` + `storageState` (Playwright docs cover this) so a logged-in session is reused across the crawl + visual passes.

## Pass A — Flow tests

For each user flow in `PROJECT.md`'s MVP slice, write one Playwright spec under `e2e/flows/`.

```ts
// e2e/flows/01-signup.spec.ts
import { test, expect } from '@playwright/test';

test('user can sign up and reach the authed shell', async ({ page }) => {
  const email = `e2e+${Date.now()}@example.com`;
  await page.goto('/login');
  // ... finish the flow (magic-link inbox check, OAuth, etc.)
  await expect(page.locator('[data-testid="user-menu"]')).toBeVisible();
});

// e2e/flows/02-core.spec.ts — the one feature your MVP exists for
// e2e/flows/03-signout.spec.ts
```

What to drive, by feature:

| Feature | What to test |
| --- | --- |
| Auth | Sign up (new email), sign in (existing), sign out, magic-link flow if used |
| MVP core slice | Full happy path from `PROJECT.md`, plus one error path |
| AI feature (sub-skill 04) | Submit input, get a typed response, check loading + error states |
| Chatbot (sub-skill 05) | Open, send message, verify reply contains a valid internal link |
| Admin dashboard (sub-skill 06) | Reject without password (401), accept with password, charts render |
| 404 / error pages | Visit `/this-does-not-exist`, confirm an on-brand 404 |

Run on localhost first, then production:

```bash
npx playwright test e2e/flows/                                     # localhost
E2E_BASE_URL=https://<your-domain> npx playwright test e2e/flows/  # production
```

## Pass B — Per-page functional crawl

This pass walks every route in `e2e/routes.ts` and asserts machine-checkable invariants. No human review needed; failures are concrete.

```ts
// e2e/crawl.spec.ts
import { test, expect } from '@playwright/test';
import { ROUTES } from './routes';

for (const route of ROUTES.filter(r => !r.auth)) {
  test(`crawl: ${route.path}`, async ({ page }) => {
    const consoleErrors: string[] = [];
    const failedRequests: string[] = [];
    page.on('console', m => { if (m.type() === 'error') consoleErrors.push(m.text()); });
    page.on('requestfailed', r => failedRequests.push(`${r.method()} ${r.url()} ${r.failure()?.errorText}`));
    page.on('response', r => { if (r.status() >= 400) failedRequests.push(`${r.status()} ${r.url()}`); });

    const response = await page.goto(route.path);
    expect(response?.status(), 'page must return 2xx').toBeLessThan(400);
    await page.waitForLoadState('networkidle');

    // Internal-link check: every <a href> on the page resolves.
    const hrefs = await page.$$eval('a[href]', els =>
      Array.from(new Set(
        (els as HTMLAnchorElement[])
          .map(a => a.getAttribute('href')!)
          .filter(h => h && !h.startsWith('http') && !h.startsWith('mailto:') && !h.startsWith('tel:') && !h.startsWith('#'))
      ))
    );
    for (const href of hrefs) {
      const r = await page.request.get(href);
      expect(r.status(), `internal link ${href} on ${route.path} must resolve`).toBeLessThan(400);
    }

    // Every visible, non-disabled button is enabled (cheap reachability check).
    const buttons = await page.locator('button:not([disabled]):visible').all();
    for (const btn of buttons.slice(0, 20)) {
      await expect(btn).toBeEnabled();
    }

    expect(consoleErrors, 'no console errors').toEqual([]);
    expect(failedRequests, 'no failed requests').toEqual([]);
  });
}
```

Run on at least desktop + mobile-lg:

```bash
npx playwright test e2e/crawl.spec.ts --project=desktop --project=mobile-lg
```

What this catches: 404s, 500s, broken internal links, JS exceptions, missing assets, network failures, disabled buttons that should be enabled. What it doesn't catch: anything visual — that's Pass C.

## Pass C — Per-page visual audit

Take a full-page screenshot of every route, on every breakpoint, then **look at every screenshot** and fill in the rubric below. This is where layout regressions hide.

```ts
// e2e/visual.spec.ts
import { test } from '@playwright/test';
import { ROUTES } from './routes';

for (const route of ROUTES.filter(r => !r.auth)) {
  test(`visual: ${route.path}`, async ({ page }, testInfo) => {
    await page.goto(route.path);
    await page.waitForLoadState('networkidle');
    await page.screenshot({
      path: testInfo.outputPath(`${route.name}.png`),
      fullPage: true,
    });
  });
}
```

Run across all five breakpoints:

```bash
npx playwright test e2e/visual.spec.ts \
  --project=mobile-sm --project=mobile-lg --project=tablet --project=desktop --project=wide
```

Screenshots land in `test-results/`. Now **open every one** and grade it against this rubric. You can read images.

### Per-page rubric — fill in for each route × breakpoint

| Criterion | ✅ / 🟡 / ❌ | Evidence (1 sentence) |
| --- | --- | --- |
| **Spacing** — consistent vertical rhythm; no cramped zones; no oceanic gaps |  |  |
| **Text wrap** — no awkward orphans, no mid-word breaks, no overflow past container |  |  |
| **Visual complexity** — ≤ 1 primary CTA per viewport; ≤ 3 distinct font weights; ≤ 5 colors visible |  |  |
| **Hierarchy** — eye lands on the intended element first; secondary content visibly secondary |  |  |
| **Alignment** — shared baseline / grid; no off-grid elements |  |  |
| **Density** — matches platform type from sub-skill 02 §1 (marketing = airy, app = dense) |  |  |
| **Mobile parity** — same content on mobile, no horizontal scroll, tap targets ≥ 44×44 |  |  |
| **Empty / error / loading states** — captured (forced via mock or throttle) and look intentional |  |  |
| **Brand consistency** — colors, type, logo, voice match the design system from sub-skill 02 |  |  |

Any 🟡 or ❌ becomes a fix. Propose each one to the user before applying:

> *"On `/dashboard` at the 768px breakpoint, the 'Create' button overlaps the search bar (Alignment: ❌). I'd like to wrap them in a flex column on screens narrower than `md`. OK?"*

Fix, redeploy if needed, re-run the relevant test. Repeat until every page × breakpoint has all ✅.

### Force the empty / error / loading states

For each core page that has them, force the state in Playwright (mock the API to return `[]`, force a 500, throttle the network to 3G) and screenshot. Don't ship with screenshots only of the happy path.

```ts
// Example: empty-state screenshot for a list page
test('dashboard empty state', async ({ page }, testInfo) => {
  await page.route('**/api/items', r => r.fulfill({ json: [] }));
  await page.goto('/dashboard');
  await page.screenshot({ path: testInfo.outputPath('dashboard-empty.png'), fullPage: true });
});
```

## Anti-patterns to avoid

- **Snapshot tests with no visual review.** A green test that no human looked at will let a broken layout ship. Always look at the screenshots.
- **Skipping breakpoints.** "It works on my laptop" ships broken mobile experiences. The five-project config above isn't optional.
- **Hardcoded selectors that depend on implementation detail.** Prefer `getByRole`, `getByLabel`, `getByTestId`. Add `data-testid` attributes if you need them.
- **Flaky timing.** Use `expect(locator).toBeVisible()` — never `setTimeout` / `waitForTimeout` to "let things settle."
- **Letting `e2e/routes.ts` go stale.** Add a comment in the route manifest reminding to update it whenever a page lands. The accessibility scan (sub-skill 09) and both crawl + visual passes share this manifest — staleness silently shrinks coverage.
- **Skipping authed routes.** Set up `storageState` and run them too.

## Exit criteria

- **Pass A (flow tests):** green on localhost AND production.
- **Pass B (functional crawl):** green on every route in `e2e/routes.ts`, on at least `desktop` + `mobile-lg`.
- **Pass C (visual audit):** rubric filled in for every route × breakpoint, every cell ✅, screenshots saved as evidence under `test-results/`.
- A `# E2E` section in `PROJECT.md` lists the tested flows, the routes crawled, the breakpoints, and the date of the last clean run.

Move on to `17-ship-checklist.md`.
