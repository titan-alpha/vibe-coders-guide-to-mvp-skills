# Modes

Not every project needs all 17 sub-skills. **Mode selection happens before bootstrap** so the agent and user agree on scope before any work starts.

## How to pick a mode

Ask the user three questions, one at a time. Then map the answers to a mode and propose it.

### Q1 — Who's going to use v1?

- **(a)** Just me / a few friends I'll show in person.
- **(b)** 5–50 people I know personally and want to onboard one-by-one.
- **(c)** Strangers from the internet (open signup, growth-oriented).
- **(d)** Paying customers from day 1.

### Q2 — What's the time horizon?

- **(a)** A weekend (24–72h). I want something demoable Monday.
- **(b)** 1–2 weeks. I want something I'd be proud to share.
- **(c)** A month or more. I want something that can carry real users.

### Q3 — What do you most need to validate?

- **(a)** Does the basic idea work? *(does anyone want this?)*
- **(b)** Will people actually use it repeatedly? *(retention)*
- **(c)** Will people pay? *(revenue)*
- **(d)** Can it scale, look professional, or pass an investor's sniff test?

## Mode catalog

For each mode: the ordered skill list, what it skips, and why.

### `quick-ship`

**Picked when**: Q1=a, Q2=a, Q3=a. *("Show me, this weekend, just to prove the idea.")*

**Skills (5)**: `01-discover`, `02-design`, `05-ai-integration` (only if the product needs AI), `13-deploy`, `16-ship-checklist`.

**Skips**: compliance, auth, admin, monetization, accessibility, security, performance, data-optimization, domain, e2e-testing, deliverables.

**You are giving up**:
- No accounts (anyone can use it; nothing is saved per user).
- No legal documents (don't show this to strangers without picking a different mode first).
- No accessibility, security, or performance guarantees.
- No automated tests.

**Right move when**: you have a hunch and want to feel it as a real product before investing more. Anyone you show should know it's a prototype.

---

### `content-site`

**Picked when**: project is a blog, docs site, marketing landing, portfolio, or content-led product. Audience is "anyone with a browser." Q3 typically (b) retention or revenue via ads.

**Skills (8–10)**: `01-discover`, `02-design`, `09-accessibility`, `11-performance`, `13-deploy`, `15-e2e-testing`, `16-ship-checklist`, plus optionally `08-monetization` (AdSense path), `14-domain`.

**Skips**: compliance (no signup → minimal surface), auth, admin, AI (optional), chatbot, data-optimization, deliverables.

**You are giving up**:
- No accounts. The content-site mode assumes nothing is gated.
- No founder admin dashboard (you don't have users to manage).

**Right move when**: the product *is* the content; the metric is pageviews and dwell time, not signups.

---

### `beta-with-users`

**Picked when**: Q1=b or c, Q2=b or c, Q3=a or b. *("I want a small, real group using this for feedback.")*

**Skills (12)**: `01-discover`, `02-design`, `03-compliance`, `04-auth`, `07-admin-dashboard`, `09-accessibility`, `10-security`, `11-performance`, `13-deploy`, `14-domain`, `15-e2e-testing`, `16-ship-checklist`. Add `05-ai-integration` and/or `06-chatbot` if the product needs them.

**Skips by default**: monetization (free beta), data-optimization (premature for first 50 users), deliverables (not pitching yet).

**You are giving up**:
- No revenue from this beta. Money comes after you've validated the product.
- No pitch deck or financial model — add the `investor-ready` mode later when you need them.

**Right move when**: you want to invite a moderated group, watch how they use it, fix what's wrong, then open more widely.

---

### `full-mvp`

**Picked when**: Q1=c or d, Q2=c, Q3=any. *("This is real and I want to do it right.")*

**Skills (all 17)**: every sub-skill, in numbered order.

**Skips**: nothing.

**Right move when**: you've already validated demand (or you're confident enough to skip that step) and you're building the v1 you'll point at users, investors, and your future self.

---

### `investor-ready`

**Picked when**: Q1=d, Q3=c or d. *("I'm raising capital or shipping with a deck.")*

**Skills (all 17, with `17-deliverables` not optional)**: same as `full-mvp`, but `17-deliverables` is mandatory and gets extra time. Pitch deck, investor one-pager, marketing one-pager, financial model, ad creative, launch announcement — all generated into `deliverables/`.

**Skips**: nothing.

**Right move when**: the product needs to look like a company, not a side project — pitch meetings, customer demos, press, fundraising.

---

## Mapping the answers

Use this as a default — confirm with the user, don't impose.

| Q1 | Q2 | Q3 | Suggested mode |
| --- | --- | --- | --- |
| a | a | any | `quick-ship` |
| any | any | content site | `content-site` |
| b or c | b or c | a or b | `beta-with-users` |
| c or d | c | any | `full-mvp` |
| d | c or d | `investor-ready` |

If two modes look plausible, propose the simpler one and explain what the more complex mode would add. Let the user pick.

## Switching modes mid-project

Allowed and expected. A common trajectory:

```
quick-ship → beta-with-users → full-mvp → investor-ready
```

When the user wants to switch, append a new "Mode change" line to `STATE.md` with the date and reason, then walk the *additional* skills the new mode requires. Skills already done at a previous mode don't re-run unless the user asks.

## Picking sub-skills inside a mode

Within a mode, the **order of skills is fixed** by their dependencies (compliance before auth, auth before admin, etc.). The agent walks them in the listed order, top to bottom.

Inside a sub-skill, the agent may still skip *sections* if the user says no (e.g., decline AdSense in `08-monetization`). That's normal — modes select skills, dialogue selects sections.

## Rule for the agent

When you select a mode, write it to `STATE.md` immediately, list the ordered skill plan, and **read STATE.md at the start of every session** so you remember what you're doing and what's already done. STATE.md is the source of truth for project progress.
