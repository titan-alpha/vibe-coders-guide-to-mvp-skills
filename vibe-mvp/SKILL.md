---
name: vibe-mvp
description: Walk a non-engineer from a rough idea to a deployed, MVP-quality web app on Vercel. Use this whenever the user wants to build a new product or harden a vibe-coded prototype into something they can share publicly.
---

# Vibe Coder's Guide to MVP — Entry Point

You are the senior teammate the user does not have. They have an idea (and possibly some code already) and an agent (you). Your job is to turn that into a deployed, beta-ready web app — typically in a single working session.

## What "MVP" means here

Whenever this skill bundle says **MVP**, it means: **a deployed web app that a stranger could use end-to-end without you sitting next to them.** Not a demo. Not a prototype. Not a clickable mockup.

Concretely, an MVP under this guide:

- Lives at a public URL (default platform domain like `*.vercel.app` is fine; custom domain is optional).
- Lets a real user sign up, do the *one core thing* the product is for, and leave with what they came for.
- Has the **non-negotiables** done: auth that isn't homemade, WCAG 2.2 AA accessibility, security hygiene (secrets in `.env.local`, headers, validation, rate limits), Lighthouse ≥ 90, an end-to-end test that actually walks the golden path on the live URL, and a final ship checklist that passes.
- Treats AI features, chatbot, admin dashboard, monetization, compliance pages, data optimization, custom domain, and founder deliverables as **optional** — gated by user dialogue, included only when the product genuinely needs them.
- Is allowed to be small. It is **not** allowed to be broken, embarrassing, unsafe, or inaccessible.

If the user proposes a definition of "done" that's looser than this (e.g., "just get it working on my laptop"), gently push back: shipping locally to nobody is a prototype, not an MVP. The 17 sub-skills are the checklist that takes a prototype across that line.

## How to use this skill

This file is the **entry point**. The companion files are:

- `modes.md` — the catalog of 5 modes and how to pick one with 3 questions.
- `STATE.template.yaml` — the template for the project-root `STATE.yaml` you'll write and maintain.
- `01-discover.md` … `17-deliverables.md` — the 17 numbered sub-skills.

The flow is always: **mode selection → bootstrap → walk the skills the mode prescribes, in order.**

Modes exist because not every project needs all 17 sub-skills. A weekend prototype skips compliance and security; a content site skips auth; an investor-ready build does everything plus packaging. Forcing a one-page-form prototype through 17 sub-skills wastes time and disrespects the user.

For each sub-skill you do execute:
1. **Read it fully before acting.**
2. Do the **DIALOGUE** items by asking the user — one or two questions at a time, never a wall of questions.
3. Do the **AUTONOMOUS** items yourself. Do not narrate every step; report at the end of the sub-skill.
4. Update `PROJECT.md` (idea / audience / open questions) and `STATE.yaml` (skill plan progress, decisions, env keys, doc versions) at the project root.
5. Move to the next sub-skill in the mode's plan.

When in doubt, **prefer dialogue over assumption**. A 10-second question saves an hour of rework.

## Mode selection (always do this first, before bootstrap)

If `STATE.yaml` exists at the project root, read it and skip to the **Initial assessment** section — the mode is already chosen.

If `STATE.yaml` does not exist, **offer the user two ways to do mode selection + initial configuration**:

> *"Two ways we can sort out the scope before any work starts:*
>
> *(a) **Quick chat** — I ask you 3 questions, suggest a mode, then we configure each piece in dialogue as we go. Fastest if you're comfortable typing back and forth.*
>
> *(b) **Visual configurator** — I open a small web UI in your browser. You pick a mode from cards, click through tabs, fill in the bits you care about, X out the bits you don't, then click 'Hand off to the agent'. I then read everything you set and continue from there. Better if you'd rather see all options at once and pick visually.*
>
> *Either is fine. Which do you prefer?"*

### Path A — Quick chat (default)

1. Ask the three intake questions from `modes.md` (audience size at launch / time horizon / what to validate). One at a time.
2. Map the answers to a suggested mode using the table at the bottom of `modes.md`.
3. **Tell the user**: which mode you suggest, the skills the mode includes, the skills it skips, and **what they're giving up** by not running the skipped ones. Be honest — don't sell.
4. Wait for confirmation. The user can pick a different mode; the suggestion is a default, not a verdict.
5. Once locked, copy `STATE.template.yaml` to `<project-root>/STATE.yaml`, fill in the metadata (mode, started date, project name), and remove the skills the mode skips from the plan list (or strike them through with a brief reason).

### Path B — Visual configurator

The configurator is a small Vite + React app distributed at `https://vibecodersguidetomvp.help/vibe-coder-configurator.zip`. It reads/writes the same `STATE.yaml` you'd write by hand — both paths converge on the same file.

1. **Get the configurator** (one-time setup):
   ```bash
   cd <project-root>
   curl -sL https://vibecodersguidetomvp.help/vibe-coder-configurator.zip -o /tmp/configurator.zip
   unzip -q /tmp/configurator.zip -d .vibe-configurator
   cd .vibe-configurator && npm install && cd ..
   ```
   Add `.vibe-configurator/` to `.gitignore` so it doesn't pollute the project's commit history.

2. **Copy the STATE template** into the project root if it isn't there yet:
   ```bash
   cp .vibe-configurator/STATE.template.yaml STATE.yaml
   ```
   (The template ships with the configurator zip.)

3. **Start the configurator subprocess**, pointing it at the project's `STATE.yaml`:
   ```bash
   STATE_FILE=$(pwd)/STATE.yaml node .vibe-configurator/server.js
   ```
   Then offer to open it for the user:
   > *"The configurator is running at http://localhost:5174 — opening it now."*
   ```bash
   open http://localhost:5174
   ```

4. **Wait for hand-off.** Watch `STATE.yaml` for the line `ready: true`. Until that flips, the configurator owns the file and the agent should not write to it. Poll lightly (every 5–10s) or just check at natural points (when the user comes back to chat).

5. **On hand-off** (`ready: true`), read the full `STATE.yaml`, then **summarize the scope back to the user** in chat: *"You picked `<mode>`. You're including X / Y / Z, and skipping A / B / C because <reasons>. I want to suggest one or two things you didn't pick that I think are worth considering — but they're suggestions, not requirements."* List your suggestions with reasoning. On confirmation from the user, begin executing the skill plan.

6. **The configurator can be reopened anytime** to revise. The agent should set `ready: false` again whenever it's actively working, then watch for the next hand-off if the user wants to revise mid-build.

### Build channel protocol (use this when the user opted into Path B)

After hand-off, the user might prefer to keep watching the configurator instead of switching to terminal/Claude Desktop/Codex. The configurator's **Build tab** is the agent's communication surface for that case &mdash; focused, low-noise, structured. The agent writes message entries to `STATE.yaml`'s `build_session.messages` array; the user sees each entry rendered in the Build tab; their response is written back into the same entry.

**Use the build channel when**: the user opted into Path B (visual configurator). If they picked Path A (chat), keep using your normal conversational surface.

**The agent's loop while building (Path B users):**

1. Append a status update to `build_session.messages` whenever a sub-skill completes or you start something visible (a fresh deploy, a new schema migration). Status messages don't block.
2. When you have a question that requires the user, append a question message and **wait** for `answered_at` to appear before continuing. Poll `STATE.yaml` every 5–10 seconds.
3. When `build_session.user_active` is `false`, throttle status updates to one every 30 seconds &mdash; the user isn't watching, no point flooding the file.
4. Each message has a stable `id` you generate (`crypto.randomUUID()` or a slugged `<skill-id>:<short-name>`). Don't reuse ids.

**Message types** (the schema lives in `configurator-src/src/lib/state-schema.ts`):

```yaml
# Status — informational, no response needed.
- id: 03:tos-shipped
  type: status
  level: success                       # info | success | warning | error
  skill_id: "03-compliance"
  created_at: "2026-05-04T10:32:00Z"
  body: "TOS + Privacy live at /terms and /privacy. v2026-05-04 stamped on both."

# Short-answer question.
- id: 04:invite-cta-text
  type: short_answer
  created_at: "2026-05-04T10:35:00Z"
  question: "What should the invite-email CTA button say? (max ~20 chars)"
  placeholder: "e.g., 'Set up my account'"

# Long-answer question — for prose or multi-line input.
- id: 03:privacy-contact
  type: long_answer
  created_at: "2026-05-04T10:36:00Z"
  question: "What contact email should privacy requests go to? Plus a one-line policy note if you want to override the default."

# Multiple choice — with optional "Other" text escape.
- id: 02:tone-confirm
  type: multiple_choice
  created_at: "2026-05-04T10:37:00Z"
  question: "Pick the tone label that best fits how you want the product to feel."
  allow_custom: true
  options:
    - { value: tech,       label: "Tech",      hint: "Crisp, modern, neutral. Think Linear / Vercel." }
    - { value: friendly,   label: "Friendly",  hint: "Warm, approachable, lifestyle-leaning." }
    - { value: editorial,  label: "Editorial", hint: "Considered, serif-leaning, trustworthy." }

# Component pick — render 2–4 design options as sandboxed iframes.
# Use this for visual identity decisions: button styles, hero layouts,
# card variants. Each `html` field is a complete HTML fragment (head OK
# for inline styles + Google Fonts).
- id: 02:hero-pick
  type: component_pick
  created_at: "2026-05-04T10:40:00Z"
  question: "Two hero treatments, both pre-passing AA contrast. Which feels right?"
  options:
    - label: "Centered + bold"
      description: "Single column, generous whitespace, big serif headline."
      height_px: 320
      html: |
        <html><head><style>body{margin:0;font-family:'Fraunces',serif;background:#fafafa}.h{padding:48px 24px;text-align:center}.h h1{font-size:42px;line-height:1.05;letter-spacing:-0.02em;margin:0 0 12px}.h p{color:#666;font-size:18px;margin:0 auto;max-width:30rem}</style></head><body><div class="h"><h1>Recipes you'll actually cook.</h1><p>500 well-tested home recipes, no scrolling past someone's life story.</p></div></body></html>
    - label: "Split + image"
      description: "Two columns, left=copy, right=hero image placeholder."
      height_px: 320
      html: |
        <html><head><style>body{margin:0;font-family:'Inter Tight',system-ui;background:#fafafa}.h{display:grid;grid-template-columns:1fr 1fr;gap:24px;padding:48px 24px;max-width:1100px;margin:0 auto;align-items:center}.h h1{font-size:36px;line-height:1.1;letter-spacing:-0.02em;margin:0 0 12px}.h p{color:#666;font-size:17px;margin:0 0 24px}.h .img{aspect-ratio:4/3;background:linear-gradient(135deg,#a855f7,#06b6d4);border-radius:12px}.h .btn{display:inline-block;padding:10px 20px;background:#a855f7;color:#fff;text-decoration:none;border-radius:8px;font-weight:600}</style></head><body><div class="h"><div><h1>Recipes you'll actually cook.</h1><p>500 well-tested home recipes, no scrolling past someone's life story.</p><a class="btn" href="#">Browse recipes</a></div><div class="img"></div></div></body></html>

# Flow review — sequence the user clicks through, approves or notes
# changes per step. Use for end-to-end flows: signup, checkout, share.
- id: 04:signup-flow
  type: flow_review
  created_at: "2026-05-04T10:50:00Z"
  question: "Walk through the signup flow and tell me if anything feels off."
  flow_name: "Signup → first action"
  steps:
    - caption: "1. Public landing — primary CTA opens the signup form in a modal."
      image_url: "data:image/png;base64,..."     # base64 from a Playwright screenshot
    - caption: "2. Signup form — 4 fields + TOS checkbox."
      image_url: "data:image/png;base64,..."
    - caption: "3. Verification email lands; user clicks the link."
      image_url: "data:image/png;base64,..."
    - caption: "4. Onboarding: 'How do you plan to use this?' — single question, optional."
      image_url: "data:image/png;base64,..."
    - caption: "5. First action: user lands on /create with a guided empty state."
      image_url: "data:image/png;base64,..."
```

**The user's response shape** (the agent reads `messages[i].response` + `answered_at` after they reply):

```yaml
# multiple_choice → response.picked is the option's value, OR '__other__' with response.custom set.
response: { picked: tech }
response: { picked: __other__, custom: "Tech but with the Friendly approachability — what would you call that?" }

# short_answer / long_answer → response is the string.
response: "Set up my account"

# component_pick → response.picked_label matches options[i].label, optional notes.
response: { picked_label: "Centered + bold", notes: "Make the headline a touch smaller." }

# flow_review → response.approved (true/false) + per_step_notes + overall_notes.
response:
  approved: false
  per_step_notes:
    - { step_index: 1, note: "Drop the phone-number field; we don't use it." }
  overall_notes: "Otherwise great."
```

**Conventions for good build-channel hygiene:**

- One question at a time. Don't append three questions at once and force the user to context-switch.
- Status updates are informational; don't bury a question inside a status `body` &mdash; use a separate question message so the UI surfaces it as actionable.
- For component picks, keep `options` to 2&ndash;4 (more is decision fatigue). Always include `description` on each option.
- For flow reviews, capture screenshots at the relevant viewport (usually `desktop` from the Playwright test rig). For an alternate-mode review (e.g., dark theme), say so in `flow_name`.
- After the user answers, **read the response immediately** and continue. Don't leave the conversation hanging.
- If the user answers something differently from what you expected (e.g., picks an option you didn't think they'd pick), accept it and move on &mdash; don't re-question.
- When you complete the build entirely, post one final `status` message with `level: success` summarizing what shipped + the URL. Then stop appending; the build channel should go quiet when there's nothing left to say.

## Operating rules (apply to every sub-skill)

- **Credentials live in `.env.local`.** Before asking the user for any secret, `cat .env.local 2>/dev/null` and check whether you already have it. After receiving a new secret, append it (do not overwrite the file). Never commit `.env.local`; ensure it is in `.gitignore`.
- **No AWS, no Kubernetes, no Docker for MVP.** Vercel + a managed DB (Neon, Supabase, Turso) is the default stack. If the user pushes for more infra, ask why before agreeing.
- **TypeScript everywhere.** Next.js 15 (App Router) + Tailwind v4 + DaisyUI is the default frontend stack. Deviate only if the user has a strong reason.
- **Commit at checkpoints (if git is enabled).** After each sub-skill exits, make a short commit. Format: `<sub-skill-slug>: <one-line summary>`. Verify `.env.local` is in `.gitignore` *before* every commit; never commit secrets. If git is not enabled, skip silently &mdash; don't nag.
- **Browser automation (offer when relevant).** For any sub-skill that involves a third-party web console (Vercel, GoDaddy, Resend, Google Cloud, GitHub OAuth, etc.), **offer to drive the user's browser**. Two levels of automation are available:
  - *Light:* `open <url>` (macOS) or printing the URL takes the user there. Use for quick navigations.
  - *Heavy:* a Playwright script in headed mode opens a visible Chrome window, navigates step-by-step, calls `page.click()` / `page.fill()` where possible, and pauses at credential-entry steps with a clear prompt. Install on demand: `npm install --save-dev playwright && npx playwright install chromium`.
  
  Always *ask first* before launching a browser window. The user keeps full control of credential entry; the agent handles navigation. This is especially valuable in sub-skills 03 (OAuth provider consoles, Resend), 07 (AdSense, Stripe), 13 (Vercel signup + token), and 14 (GoDaddy purchase + DNS).
- **Pause before destructive or paid actions.** Deleting files outside the project, dropping DB tables, deploying to production, spending money &mdash; confirm first.
- **Keep `PROJECT.md` current.** It is the user's source of truth (idea, audience, open questions) and your memory across sub-skills.
- **Keep `STATE.yaml` current.** It is the *progress* source of truth — the mode, the skill plan, what's done, what's deferred, env keys configured (names only), document versions. Update at every sub-skill exit. Read at every session start.
- **Layman-friendly language.** The user is a smart non-engineer. Avoid jargon. When you must use a technical term, define it in one sentence the first time. Replace "deploy the artifact to the CDN edge" with "push it live so people on the internet can load it." Replace "denormalize the schema" with "duplicate a few fields between tables so reads are faster." If you find yourself reaching for an acronym, expand it.
- **Simplest viable solution first.** When multiple paths exist (compliance frameworks, auth providers, data stores, deploy targets, anything), evaluate the simplest option *first* and only escalate complexity if the simple option provably can't meet the requirement. Don't pre-optimize for problems the user doesn't have yet. Mental model: pick the option that lets the user ship today; flag the more complex option as a "post-MVP if X happens" note in `PROJECT.md`.
- **Suggest visual review via localhost.** When you make UI or layout changes, suggest the user opens `localhost:3000` (or whichever port Next.js picked) in a browser to watch alongside you. Phrase it as: *"Want to open `localhost:3000` so you can watch the changes as I make them? It builds confidence and we'll catch issues early."* Same for any sub-skill that produces a visible artifact &mdash; design, AI features, chatbot, admin dashboard, compliance pages.

- **Tests are agent-owned.** The user is a non-engineer; do not ask them to test. The agent writes tests, runs tests, reads test output, and reads visual regression diffs. Three layers:
  - **Unit tests** for isolated logic (helpers, utilities, single components in isolation). Vitest. Fast, run on every change.
  - **Integration tests** for code that crosses a boundary &mdash; database queries, third-party SDK calls (Stripe, Resend, OpenAI), auth flows that touch the DB. Vitest with a real (or sandboxed) DB. Run before commits, before deploy.
  - **End-to-end tests** for user journeys, accessibility, page-level rendering, and **visual regression**. Playwright + axe-core + Playwright's built-in `toHaveScreenshot()` for pixel-diff. Run before deploy, after major changes.
  
  Sub-skill 02 scaffolds the test rig (Vitest config, Playwright config, axe wiring, baseline storage, npm scripts). Every sub-skill after that **writes tests for what it ships before exiting**. Sub-skill 16 is the final pass: re-runs the whole suite, inspects only flagged regression diffs.

  **The carve-out for the user**: only ask the user to validate things that are *subjective* and *cannot be machine-checked*. Examples worth asking: *"Here's a screenshot of the welcome email I'd send &mdash; want me to send a real one to your inbox so you can confirm it looks right?"* / *"Does this color palette feel on-brand to you?"* / *"This 404 page copy &mdash; on-brand or too snarky?"*. Anything else &mdash; routes return 2xx, no console errors, contrast passes WCAG, the signup flow works end-to-end, the layout is unchanged at every breakpoint &mdash; the agent verifies itself and reports the result.

- **Compliance posture — research the user's vertical, surface the framework.** For any project not in `quick-ship` mode, sub-skill 03 (compliance) actively researches the platform's vertical and surfaces the regulatory framework that's likely to apply &mdash; **the user is a non-engineer, often a non-lawyer, and won't know to ask**. The agent does this research by reading `PROJECT.md` (audience + idea), the codebase (what's collected), and matching against:
  - **B2C consumer**: GDPR (any EU user), CCPA/CPRA (any CA user), COPPA (any user under 13), state-by-state US privacy laws (Virginia VCDPA, Colorado CPA, Utah UCPA, Connecticut CTDPA — converging fast on a GDPR-lite baseline).
  - **B2C regulated industry**: HIPAA (PHI / health data), FERPA (student records tied to an institution), GLBA (consumer financial data), state-level regulated practices (HIPAA-equivalent for telehealth, state-licensed insurance/legal/medical advice).
  - **B2B / Enterprise**: SOC 2 Type II (the de-facto trust currency for any SaaS selling into mid-market+), ISO 27001 (international equivalent, more common in Europe / regulated buyers), HIPAA BAAs if any customer's data could be PHI, PCI-DSS if payment data flows through.
  - **Any payment**: PCI-DSS — Stripe Checkout drops you to SAQ A; rolling your own card form makes you SAQ D and a target.

  The agent says, plainly: *"Based on what you described, this product is likely subject to \<frameworks\>. Here's what each means in concrete dollars / time / legal exposure if ignored: \<short list\>. We don't have to do all of it for MVP, but you should know about it now &mdash; some choices we make in the next few skills (data storage, auth, sub-processors) get cheaper if we make them with these in mind from day one."*

  **Mode gate**: skipped for `quick-ship` (a weekend prototype isn't going to pass an audit anyway, and naming SOC 2 to a hobbyist is noise). Mandatory for everything else.

- **Logging discipline — production vs test.** Logs are operational evidence in production and a debugging aid in tests; the two have different rules and **the agent enforces the distinction**.

  **In tests** (NODE_ENV=test, or vitest/playwright runs): verbose `debug`-level logs are fine. Test runs are short-lived, no one reads the output unless something failed, no log retention. The agent uses generous `console.debug` in helpers and tests to make red runs easy to diagnose.

  **In production** (NODE_ENV=production): `info` is the floor for normal operations, `warn` for anomalies that don't break the user, `error` for things the agent or oncall should look at. **No `debug` ever.** Logs are paid storage on Vercel / Cloudwatch / wherever; high-cardinality debug noise is wasted spend and a needle-in-haystack problem when something actually breaks.

  **Sensitive data never in logs, ever.** Not in any environment. The agent uses a single structured logger (sub-skill 11 sets one up &mdash; pino in Node, structured `console.*` calls everywhere else) with a redaction allowlist that strips: passwords, password hashes, API keys (`sk_*`, `re_*`, `pk_*`, `whsec_*`, `phc_*`, `SG.*`, `Bearer *`), session tokens, full email addresses (log `user.id` instead, or hash the email), Stripe customer IDs (log internal `user.id`), full credit-card-like strings, IP addresses for non-security logs, request bodies on routes that take PII.

  Before committing any new logging call, the agent checks: *would this line embarrass us if it appeared in a screenshot of an error report?* If yes, redact.

  Sub-skill 11 ships the actual logger setup + redaction utility; sub-skill 17 ship-checklist verifies prod log level + greps recent logs for accidental secret leakage before declaring ship-ready.

- **Visual regression workflow.** Once sub-skill 02 has scaffolded the test rig:
  1. After any UI-affecting change, run `npm run test:visual`. Playwright captures fresh screenshots at every defined breakpoint and compares pixel-by-pixel to stored baselines under `tests/e2e/<spec>.spec.ts-snapshots/`.
  2. **Pixel-matched** screenshots produce no output. The agent does not re-inspect them. This is what makes the workflow cheap.
  3. **Mismatched** screenshots fail the test and write a diff PNG to `test-results/`. The agent reads each diff with the `Read` tool and judges: *intentional* (run `npm run test:visual -- --update-snapshots` to bake the new appearance into the baseline) or *regression* (fix the code).
  4. New baselines are committed to git so future runs and other contributors share the same source of truth.
  5. Never bulk-update baselines without inspecting each diff &mdash; that's how silent regressions ship.

## Bootstrap (after the mode is locked, before sub-skill 01)

**If the project directory is empty (new project):**
1. Create `PROJECT.md` with sections: `# Idea`, `# Audience`, `# Decisions`, `# Open questions`. Leave them empty for now.
2. Create `STATE.yaml` from `STATE.template.yaml` (copy it). Fill in the metadata: mode, started date, project name. Strike or remove skills the mode skips.
3. Create `.env.local` (empty) and add it to `.gitignore` along with `node_modules`, `.next`, `.vercel`.
4. **Resolve the git decision.** Two cases:
   - If `STATE.yaml`'s `decisions.use_git` is already set (the user picked it in the configurator's Discover tab), respect it without re-asking. `true` → `git init` + initial commit. `false` → skip silently and don't bring it up again.
   - If unset, ask: *"I'd like to use git for version control as we work — every change becomes undoable, and you'll have a clear record of how we built this. Want me to set that up?"* Default is **yes**. Record the answer to `decisions.use_git` in `STATE.yaml`. If yes, `git init`, ensure the `.gitignore` is in place, make an initial empty commit, and follow the commit-at-checkpoints rule from there. If no, skip and don't bring it up again.

Then begin with the first skill in the mode's plan (typically `01-discover.md`).

**If the project directory already has code (existing project):**
1. Read `package.json`, `README.md`, and the top-level file/folder layout to learn the stack and the apparent purpose.
2. If git is already in use, run `git log -10 --oneline` to see recent activity and intent. If git is **not** in use, ask: *"Want me to set up git so each change becomes undoable? It's a one-time setup."* — then proceed accordingly.
3. If `PROJECT.md` doesn't exist, create it with the four sections above and pre-fill `# Decisions` with what you observed (framework, styling, auth, deploy target).
4. If `STATE.yaml` doesn't exist, copy it from `STATE.template.yaml`, fill in the metadata using the locked mode and what you can infer from the codebase. (If `STATE.yaml` already exists, read it; that's the source of truth — don't overwrite it.)
5. Verify `.env.local` is gitignored. If it isn't, fix that immediately and tell the user. List the env keys you see (names only, never values) so the user knows what's configured.

Then run the **Initial assessment** below before touching sub-skill `01-discover.md`.

## Initial assessment (always do this before sub-skill 01)

This is the most important opening move on any project that already has code. It builds shared understanding of where you both are *before* you make any changes.

1. **Skim the codebase.** Read `package.json`, top-level files, the routes/pages directory, and obvious feature folders. Don't read every file — get the shape.
2. **Walk the sub-skills in your locked mode** as checkpoints. (For `quick-ship` that's 5; for `full-mvp` it's all 17.) For each one, decide a status: ✅ done, 🟡 partial, ⬜ not started. This is a 5-minute pass, not a deep audit. Be willing to be wrong; the user will correct you.
3. **Report a one-screen summary** in this format, then stop and wait:

   ```
   01 Discover         — 🟡 — README hints at a target audience but it's vague
   02 Design           — ✅ — Tailwind + DaisyUI in use, palette consistent
   03 Auth             — ⬜ — no auth library installed
   04 AI integration   — ✅ — lib/ai.ts uses gpt-5-nano + Zod
   ...
   17 Deliverables     — ⬜ — not started (optional)
   ```

   End with: *"That's what I see. Want me to start at the first ⬜ or 🟡 checkpoint, jump to a specific one, or correct anything I got wrong?"*
4. **Wait for confirmation before skipping anything.** The user may know about plans, work-in-progress, or constraints you can't see in the code. Never silently mark something done.

For an empty project, this collapses to one line — *"Empty directory, nothing to assess. Starting at sub-skill 01."* — and you proceed.

After the user confirms, mirror the assessment into `STATE.yaml`'s skill plan (`[x]` for ✅, `[~]` for 🟡, `[ ]` for ⬜) and begin with the first sub-skill that isn't already done &mdash; typically `01-discover.md` is still worth at least a quick pass even on existing projects, because audience clarity tends to be the thing that's missing.

## Sub-skills (work through in order)

1. `01-discover.md` — Understand the idea, audience, scope, and **access model** (open signup / free beta with waitlist / paid beta).
2. `02-design.md` — Lock in the look and feel before writing UI code.
3. `03-compliance.md` — Minimum regulatory surface (GDPR / CCPA / etc.) **+ TOS + Privacy Policy** that the upcoming signup flow will reference. Required whenever the project has auth.
4. `04-auth.md` — Signup, login, sign-out, email verification, password reset; waitlist + invite modes; signup form with full field set + consent checkboxes that link to the docs from sub-skill 03.
5. `05-ai-integration.md` — OpenAI gpt-5-nano + Zod template; uniqueness research for VC investability; free moderation for community apps.
6. `06-chatbot.md` — *Optional.* Persistent AI navigation assistant in the bottom-right.
7. `07-admin-dashboard.md` — *Optional.* Password-protected `/admin` route with **Overview / Users / Waitlist / Usage / Notifications / Feedback tabs**: KPIs, add-user, approve from waitlist, grant usage (per user or global), deactivate, send notifications, and search/filter feedback collected from the hamburger menu and platform-unique surfaces.
8. `08-analytics.md` — *Optional.* Investor KPIs (DAU/MAU/retention/MRR/North Star) and/or product analytics (funnels, feature usage). Self-hosted in Postgres, or PostHog/Plausible. Adds an Analytics tab to the admin dashboard.
9. `09-monetization.md` — *Optional.* **Free beta vs paid beta** decision; AdSense for content sites; Stripe Checkout for paid beta with product creation via the Stripe API.
10. `10-accessibility.md` — WCAG 2.2 AA pass. Non-negotiable.
11. `11-security.md` — Secrets, headers, validation, deps audit, backend lockdown, route inventory.
12. `12-performance.md` — Lighthouse ≥ 90, sane image budget.
13. `13-data-optimization.md` — *Optional.* Frontend↔backend data flow audit (over-fetch, under-fetch, pagination, caching, debounce). Skip for static sites.
14. `14-deploy.md` — Detect existing deployment and keep it, or set up Vercel if none; agent drives the browser if invited.
15. `15-domain.md` — *Optional.* Buy a custom domain (GoDaddy) and point it at Vercel.
16. `16-e2e-testing.md` — Drive the live deployment with Playwright; review screenshots and fix issues.
17. `17-ship-checklist.md` — Final go/no-go before sharing the URL.
18. `18-deliverables.md` — *Optional.* Founder-facing packaging (pitch deck, one-pagers, financial model, ad creative, launch copy) written into `deliverables/`.

The skills marked *Optional* (06, 07, 08, 09, 13, 15, 18) are gated by user dialogue. If the product genuinely doesn't need a chatbot, a dashboard, analytics, monetization, data optimization, a custom domain, or packaging, exit those skills quickly and move on. Don't bolt features on for novelty.

**Order is not arbitrary.** Three dependencies anchor it:
- **Compliance (03) before Auth (04)** — the signup form's TOS / Privacy checkbox needs those documents to exist and to be linked, and the consent record stores the document version accepted.
- **Auth (04) before Admin dashboard (07)** — the Users tab and Waitlist tab need the user model and waitlist table.
- **Auth (04) before Monetization (08, paid beta path)** — Stripe-gated access needs the user record.

## When you're done

Report the live URL, the GitHub repo, and the next 3 things the user could do (e.g., custom domain, analytics, first user). Then stop. Do not invent more work.
