# 16 · E2E Testing — Final Regression Pass

Goal: re-run the test suite that has been built incrementally across every prior sub-skill (since the rig was scaffolded in 02), confirm everything still works at the public URL, and surface only the visual diffs that need a human eye. The user does not test; the agent does.

This sub-skill is **not** where the suite gets created. The suite was created in 02 and grew sub-skill by sub-skill — every feature added between 03 and 15 already shipped with its own unit, integration, and visual tests (per SKILL.md's "Tests are agent-owned" operating rule).

What this sub-skill does:

- **Pass A — Run the full suite.** Vitest unit + integration + Playwright e2e + axe a11y + visual regression.
- **Pass B — Functional crawl.** Walk every route in the shared manifest, assert no console errors, no broken links, every interactive element reachable.
- **Pass C — Visual regression review.** Inspect ONLY the screenshots that failed pixel-diff against their stored baselines. Never bulk-update; never re-inspect anything that pixel-matched.
- **Pass D — Production smoke.** Re-run the journey tests against the deployed URL to catch deploy-only regressions.

## DIALOGUE — confirm with the user

Browser automation will open visible windows, exercise every flow, and use a clearly-marked test account. Confirm before kicking off.

> *"I'm going to run the full test suite — unit, integration, end-to-end, accessibility, and visual regression. Most of it runs without me looking at anything (it's just pass / fail). Visual regression is the one place where I'll spend a few minutes inspecting any screenshots that don't match the stored baseline. If a change was intentional I update the baseline; if it's a regression I fix it. The whole thing takes ~5 minutes the first time. OK to start?"*

If the user has data isolation concerns, ask whether to test against staging or production. For production, use a clearly-marked test account (`e2e+<timestamp>@<domain>`).

## Pass A — Full local suite

```bash
npm run test:unit
npm run test:integration
npm run test:e2e
```

Expected outcome:
- Unit + integration: green. If red, fix the code or update the test (only update if the behavior change was intentional).
- E2E flow specs: green at `localhost:3000`.
- Accessibility scan: zero violations on every route in `tests/e2e/routes.ts`. (See sub-skill 09 for the axe wiring.)

If anything is red, fix it before continuing. Never proceed to ship with red tests.

## Pass B — Functional crawl

```bash
npm run test:crawl -- --project=desktop --project=mobile-lg
```

The crawl spec (added in 02 as a template, expanded as sub-skills added routes) walks every route in `tests/e2e/routes.ts` and asserts:

- 2xx response on initial load.
- No `console.error` entries.
- No failed network requests (`status >= 400` or `requestfailed`).
- Every internal `<a href>` resolves.
- Every visible non-disabled button is reachable.

Any failure here is a regression that must be fixed before ship. Console errors are especially common after a code change touched logging or a third-party SDK upgraded.

## Pass C — Visual regression (the cheap-and-fast loop)

This is where the screenshot baselines from 02 onward earn their keep.

```bash
npm run test:visual
```

Playwright captures a fresh screenshot for every test × project (5 viewports: mobile-sm, mobile-lg, tablet, desktop, wide) and compares pixel-by-pixel against the stored baselines under `tests/e2e/visual.spec.ts-snapshots/`.

**Three outcomes:**

1. **All matched** — nothing to inspect. The agent reports "✅ visual regression: 47 / 47 baselines matched" and moves on. **Do not** open the screenshot files; do not narrate what you would have seen. Pixel-match is the contract; trust it.

2. **Some mismatched** — Playwright writes the failing test name plus three artifacts to `test-results/`: the baseline (`*-expected.png`), the new render (`*-actual.png`), and the diff (`*-diff.png`). The agent **reads each diff PNG** with the `Read` tool. For each one, judge:

   | What the diff shows | Action |
   | --- | --- |
   | Intentional change (a feature you just shipped that legitimately altered the page) | Update the baseline: `npm run test:visual:update -- <test name>`. **Inspect the new baseline once before committing** to confirm it matches expectation. Then `git add tests/e2e/visual.spec.ts-snapshots/` and commit with the reason. |
   | Layout / styling regression (something broke that shouldn't have) | Fix the code, re-run `npm run test:visual`, repeat until matched. |
   | False positive from font anti-aliasing or asynchronous animation | Tighten the spec: pre-load the font with `await page.evaluate(() => document.fonts.ready)`; disable specific animations with `animations: 'disabled'` on the screenshot call. Don't loosen the global threshold. |

3. **Mass mismatched** (≥ 30% of baselines failing) — usually means a global stylesheet, theme variable, or font URL changed. Don't update the baselines reflexively; investigate the root cause first. If the change is genuinely intended (e.g., the user redesigned the color palette), bulk-update is fine — but only after confirming the FIRST diff looks correct.

### Critical rule

**Never run `npm run test:visual:update` without inspecting the diffs first.** That's how silent regressions ship. The whole point of the workflow is that re-runs are cheap because pixel-match is trustworthy. Bulk-updating without inspection breaks that contract.

### Forcing rare states (empty / error / loading)

The visual baselines committed by 02–15 cover happy paths. Edge states need explicit specs:

```ts
// tests/e2e/visual-states.spec.ts (each sub-skill that ships a list/form should add to this)
test('dashboard empty state', async ({ page }) => {
  await page.route('**/api/items', r => r.fulfill({ json: [] }));
  await page.goto('/dashboard');
  await expect(page).toHaveScreenshot();
});

test('signup form validation error', async ({ page }) => {
  await page.goto('/signup');
  await page.click('button[type=submit]'); // submit empty
  await expect(page).toHaveScreenshot();
});
```

If 02–15 missed these, add them now and capture baselines.

### Light + dark theme coverage

Every visual test must run in BOTH theme modes (sub-skill 02 made dark+light non-negotiable). The pattern:

```ts
for (const route of ROUTES.filter(r => !r.auth)) {
  for (const theme of ['vibelight', 'vibedark'] as const) {
    test(`visual: ${route.path} (${theme})`, async ({ page }) => {
      await page.addInitScript((t) => localStorage.setItem('theme', t === 'vibedark' ? 'dark' : 'light'), theme);
      await page.goto(route.path);
      await page.waitForLoadState('networkidle');
      await expect(page).toHaveScreenshot();
    });
  }
}
```

Doubles the baseline count but catches dark-mode-only regressions (the most-common class of UI bug a single-mode test would miss).

## Pass D — Production smoke

After deploy (sub-skill 14), re-run the **flow specs** (not visual — production rendering can differ from localhost in font loading and CDN delivery, so visual baselines are localhost-only).

```bash
E2E_BASE_URL=https://<your-domain> npm run test:e2e -- tests/e2e/flows/
```

The journey tests (signup → core feature → sign-out) should pass identically. If they don't, something in the deploy environment is different (env var missing, DNS not propagated, build mode mismatched). Investigate before declaring ship-ready.

## Subjective checks the user must do (the carve-out)

Almost nothing — but a small set of decisions are subjective and need a human eye:

1. **Email rendering**: the agent has been running templates through the dev email-debug helper (sub-skill 04 step 7) which writes rendered HTML to `tmp/emails/`. Now ask: *"Want me to send the welcome email to your real inbox so you can confirm it looks right?"* Send one of each approved trigger. The user replies with approve / tweak / reject. Apply tweaks, re-send if needed.
2. **Brand vibe pass**: open the deployed site (sub-skill 14) in their browser and ask: *"Look at the landing page and the dashboard. Anything feel off-brand or unexpected?"* This is a single 30-second check; the agent doesn't enumerate every detail.
3. **One-paragraph copy review**: any user-facing prose the agent wrote without explicit user input (404 page, empty states, marketing sub-headlines) — show it once, ask: *"Anything you'd change?"*

That's the entire user-side test surface. Everything else the agent verifies.

## Anti-patterns to avoid

- **Re-inspecting matched screenshots.** Pixel-match is the contract. If you find yourself opening a passed baseline "just to be sure," you're undermining the workflow's whole purpose.
- **Bulk-updating baselines without inspecting each diff.** The single most-common way to ship a silent regression. Always read the diff PNG first.
- **Snapshot tests with no visual review on the FIRST run.** When a baseline is captured, the agent inspects that baseline once before committing. After that, the pixel-diff loop takes over.
- **Running `--update-snapshots` to "fix" a flaky test.** Flakiness is a sign of an underlying problem (font loading, animation, network timing). Stabilize the test, don't paper over it.
- **Visual regression against the production URL.** Localhost only. Production renders can shift due to CDN font caching and don't represent the source-of-truth render.
- **Asking the user to test something the agent could have verified.** The user's time is spent on subjective decisions only (email rendering, brand, copy). Functional correctness is the agent's job.
- **Letting the suite go red between sub-skills.** Every sub-skill exits green or it doesn't exit. Don't accumulate failures across skills hoping to fix them in this final pass.
- **Skipping dark-mode visual coverage.** Both themes ship. Both themes get baselined. Half the visual budget for either-theme-only-mode is the wrong answer.

## Exit criteria

- `npm run test` is green: unit + integration + e2e all pass.
- `npm run test:crawl` is green on every route in `tests/e2e/routes.ts`, on at least desktop + mobile-lg.
- `npm run test:visual` is green in BOTH light and dark theme — all baselines matched, OR all diffs were inspected and either fixed or rebaselined with the reason captured in the commit message.
- Production smoke (`E2E_BASE_URL=<prod> npm run test:e2e -- tests/e2e/flows/`) passes.
- The 1–3 subjective checks for the user (email rendering, brand vibe, copy) have been completed and approved.
- A `# E2E` section in `PROJECT.md` lists the date of the last clean run, the routes covered, the breakpoints, and any rebaselines committed (with reason).

Move on to `17-ship-checklist.md`.
