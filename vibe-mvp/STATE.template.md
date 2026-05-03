# Project State

This file is the agent's source of truth for project progress. It lives at the **project root** (not in the skills directory). Read it at the start of every session before doing any work; update it after every sub-skill exits.

If you are an agent and you don't see this file at the project root, copy this template to `<project-root>/STATE.md` and fill in the metadata.

---

**Mode**: `<mode-name>`  *(see `modes.md` — quick-ship / content-site / beta-with-users / full-mvp / investor-ready)*
**Started**: `<YYYY-MM-DD>`
**Last updated**: `<YYYY-MM-DD>`
**Project name**: `<the working title>`
**Public URL**: `<*.vercel.app or custom domain, or "not deployed yet">`

## Skill plan (in order, per mode)

Mark each skill `[ ]` pending, `[~]` in progress, `[x]` done. Append a one-line note when you mark it done — what was decided, what was deferred.

- [ ] 01 Discover
- [ ] 02 Design
- [ ] 03 Compliance
- [ ] 04 Auth
- [ ] 05 AI Integration
- [ ] 06 Chatbot
- [ ] 07 Admin Dashboard
- [ ] 08 Monetization
- [ ] 09 Accessibility
- [ ] 10 Security
- [ ] 11 Performance
- [ ] 12 Data Optimization
- [ ] 13 Deploy
- [ ] 14 Domain
- [ ] 15 E2E Testing
- [ ] 16 Ship Checklist
- [ ] 17 Deliverables

*(Strike out the skills your mode skips, or remove them. The remaining list is your active plan.)*

## Skipped (with reason)

- 17 Deliverables — *not in this mode (`beta-with-users`); add later if pitching investors.*

## Decisions

Append one line per locked decision. Don't delete past decisions; append "(superseded by ...)" if you change one.

- **Access model**: `<open / waitlist / paid>`
- **Auth path**: `<email+password / magic link / OAuth>`
- **Email service**: `<Resend / SendGrid / none>`
- **Stack**: `<Next.js 15 + Tailwind v4 + DaisyUI + Drizzle + Postgres (Neon)>`
- **Initial usage grant**: `<INITIAL_USAGE_GRANT value>`

## Document versions

Bump these when the doc changes. Sub-skill 04 stamps each consent record with the current version.

- Terms of Service: `<YYYY-MM-DD>`
- Privacy Policy: `<YYYY-MM-DD>`

## Env keys configured (names only — never values)

- `AUTH_SECRET`
- `RESEND_API_KEY`
- `EMAIL_FROM`
- `ACCESS_MODE`
- `INITIAL_USAGE_GRANT`
- `ADMIN_PASSWORD`
- *(and so on)*

## Sub-processors with signed DPAs

- *(populated by sub-skill 03)*

## Open questions / blockers

- *(none currently)*

## Mode changes

If the user switches modes mid-project, append a line here with the date, the new mode, and why.

- *(none yet)*

---

## Update rules (for the agent)

1. **At session start**: read this file before any tool call. Restore your understanding from it. If a memory or assumption conflicts with what's here, trust this file.
2. **At sub-skill start**: mark the line `[~]` in progress.
3. **At sub-skill exit**: mark the line `[x]` done, append a one-line note (what shipped, what deferred), update relevant sections (Decisions / Document versions / Env keys / Open questions), and bump **Last updated**.
4. **When the user changes their mind** about something: append to Decisions with the new value, and add `(superseded by Decisions entry from <date>)` to the old one.
5. **Never delete completed skills** from the plan — strike them through (`~~text~~`) only if the mode changed and a previously-done skill is now genuinely irrelevant.

This file is part of the project's working memory. It is **not a substitute for `PROJECT.md`** (idea / audience / open questions) or for code comments — it is purely *progress* state.
