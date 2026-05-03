# 04 · Auth (signup, login, verification, access modes)

Goal: pick the simplest user authentication that fits the audience and the **access model** chosen in sub-skill 01, then wire the full flow — signup, login, sign-out, password reset, and (when needed) email verification — without rolling your own session logic.

This sub-skill assumes sub-skill 03 (compliance) has already produced the Terms of Service and Privacy Policy pages. The signup form built here links to them and records consent against their version.

## DIALOGUE — confirm the access model

Look up the access model from `PROJECT.md # Decisions`. Reaffirm with the user, because it changes everything that follows:

| Access model | What this sub-skill builds |
| --- | --- |
| **Open signup** | Public signup form, anyone can register, immediate access. |
| **Free beta with waitlist** | Public waitlist form (email + optional intro). Signup is **gated** until an admin (sub-skill 07) approves the email. Approved users can self-register with a password they choose. |
| **Paid beta** | Same as open or waitlist, plus a Stripe-gated step (sub-skill 08) before the user can use the product. |

Then ask one or two more:

1. **How do you want users to sign in once they have access?**
   - **Email + password** (most common when admins want to whitelist specific emails). Requires email service for password reset and (recommended) for email verification.
   - **Magic link only** (passwordless). Friendlier; requires email service.
   - **OAuth only** (Google / GitHub). No email service needed for sign-in but you'll still want one for invites / password reset on credentials accounts.
2. **Will user data be tied to accounts?** (Determines whether a database is part of this step — yes for any waitlist/paid-beta product.)

## AUTONOMOUS — pick the stack

| User experience | Use |
| --- | --- |
| No login | Skip this skill |
| Magic link only | **Auth.js v5** Email provider + **Resend** |
| OAuth only | **Auth.js v5** with Google &/or GitHub providers |
| Email + password (with verification + reset) | **Auth.js v5** Credentials provider + Drizzle/Prisma + **Resend** |
| Phone / SMS | **Clerk** (paid; warn the user) |

Default for **free beta with waitlist or paid beta**: **Auth.js v5 Credentials (email + password) + Drizzle + Resend**, because the admin-controlled access pattern works best when each user explicitly sets a password they own (no email-roundtrip every login). Magic link is fine too if email reliability is high.

Default for **open signup**: **Auth.js v5 with magic link via Resend, plus Google as a one-tap option.** Lowest friction.

## Implementation rules (apply to every path)

- Never write your own password hashing or session storage. If you find yourself reading `bcrypt` docs, stop and reconsider — use `bcryptjs` via Auth.js's Credentials adapter, never hand-roll it.
- Walk the user through every external console (Google Cloud, GitHub OAuth, Resend) step-by-step. Don't assume they've done it before.
- Store all credentials in `.env.local`:
  ```
  AUTH_SECRET=...           # generate with `openssl rand -base64 32`
  AUTH_GOOGLE_ID=...
  AUTH_GOOGLE_SECRET=...
  AUTH_GITHUB_ID=...
  AUTH_GITHUB_SECRET=...
  RESEND_API_KEY=re_...
  EMAIL_FROM="App <onboarding@resend.dev>"   # use Resend default until a domain is verified
  ACCESS_MODE=open|waitlist|paid
  ```
- Build one `/login` page (DaisyUI `btn` + provider logos is enough) and a sign-out button visible from the authed shell. Don't invent custom auth UI in v1.
- One `auth()` server-side check per protected route. Don't sprinkle checks throughout components.

## Email service — Resend (default) or SendGrid (alternative)

If the chosen path needs email (magic link, password verification, password reset, or admin invites), set up an email provider.

Ask first:

> *"Default email service is Resend (free tier, simplest setup). If you already have a SendGrid account from another project, we can use it instead. Either works. Which do you have or prefer?"*

Before asking, run `grep -qE '^(RESEND_API_KEY|SENDGRID_API_KEY)=' .env.local 2>/dev/null && echo found`. If one is already set, skip the corresponding walkthrough.

### Resend walkthrough (default)

Walk the user through it:

1. *"Open https://resend.com/signup."*
2. *"Sign up (Google or email both work). Verify your email if asked."*
3. *"Once you're in, go to https://resend.com/api-keys."*
4. *"Click **Create API Key**. Name it after the project. **Sending Access** scope is fine. Copy the key &mdash; you won't see it again. Paste it here."*

Append to `.env.local`:
```
RESEND_API_KEY=re_...
EMAIL_FROM="App <onboarding@resend.dev>"
```

`onboarding@resend.dev` is Resend's shared sender that works immediately without verifying a domain. For production, the user should add their own domain in Resend (DNS records: SPF, DKIM, MX) and switch `EMAIL_FROM` to `noreply@<their-domain>`. That's a sub-skill 14 (domain) follow-up.

### SendGrid walkthrough (alternative)

Walk the user through it:

1. *"Open https://signup.sendgrid.com if you don't have an account."*
2. *"Sign in. Settings → API Keys → Create API Key. Choose 'Restricted Access' and grant 'Mail Send: Full Access'. Name it after the project. Copy the key."*
3. *"Sender Authentication → Single Sender Verification (fastest path) → add the email you'll send from. Verify it via the email SendGrid sends."*

Append to `.env.local`:
```
SENDGRID_API_KEY=SG.xxxxxxxxxxxxxxxxxxxxxxxx
EMAIL_FROM="App <verified-sender@example.com>"
```

### Install (pick ONE)

Install only the package for the chosen provider — this is an OR, not an AND:

```bash
npm install resend         # if Resend
npm install @sendgrid/mail # if SendGrid
```

### `lib/email.ts` — single entry point

Create `lib/email.ts` as the **single entry point** for every outbound email (signup verification, password reset, admin invite, marketing — all routed through here). It picks the implementation based on which key is set:

```ts
import { Resend } from 'resend';
import sgMail from '@sendgrid/mail';

const FROM = process.env.EMAIL_FROM ?? 'App <onboarding@resend.dev>';
const useResend = Boolean(process.env.RESEND_API_KEY);
const useSendGrid = Boolean(process.env.SENDGRID_API_KEY);

if (useSendGrid) sgMail.setApiKey(process.env.SENDGRID_API_KEY!);
const resend = useResend ? new Resend(process.env.RESEND_API_KEY!) : null;

export const emailEnabled = useResend || useSendGrid;

export async function sendEmail(args: { to: string; subject: string; html: string }) {
  if (!emailEnabled) throw new Error('Email service not configured.');
  if (useResend) {
    const { error } = await resend!.emails.send({ from: FROM, ...args });
    if (error) throw new Error(`Resend send failed: ${error.message}`);
  } else {
    await sgMail.send({ from: FROM, ...args });
  }
}
```

The exported `emailEnabled` boolean lets the admin dashboard (sub-skill 07) branch between *"send an invite email"* and *"just whitelist the address."*

## AUTONOMOUS — analyze the platform and propose email triggers

Once an email service is wired, the agent should figure out **which emails this specific product actually needs** rather than dumping a generic catalogue on the user.

### 1. Read the project context

Read `PROJECT.md` and `STATE.md` end-to-end. Pull out:

- **What the product does** (the one-line description and the core user flow).
- **Who uses it** (target audience — consumers, internal team, paid B2B, etc.).
- **Access mode** — open / waitlist / paid (from `PROJECT.md # Decisions`).
- **AI usage limits** — does sub-skill 07's `usageGranted` / `usageConsumed` model apply? (i.e., is this an AI product that meters per-user calls?)
- **Product shape** — content product (posts, comments, subscribers, digests) vs. SaaS tool vs. utility.
- **Stripe / billing** — is sub-skill 08 in scope?
- **Admin flows** — does the admin dashboard issue invites, deactivate users, grant usage?

### 2. Trigger catalog (source of truth)

Pick the triggers that fit the product. Don't propose all of them. The catalog:

- **Universal (any product with auth)**: signup confirmation; email verification (if pre-verification required); password reset; sign-in from a new device (security alert).
- **Waitlist mode**: waitlist confirmation ("we got your request"); approval invite (with one-time signup link); rejection (optional, polite, with reason).
- **Admin invite mode**: admin-issued invite ("You've been invited to join X").
- **Usage-gated AI products**: low-usage warning at 80% consumed; usage exhausted; usage refilled (when admin grants more).
- **Content products**: weekly digest of new posts; comment notification; subscriber confirmation.
- **Paid beta / Stripe**: subscription receipt; failed payment; subscription cancelled; renewal reminder (3 days before).
- **Engagement**: re-engagement after N days inactive; first-action celebration ("you did your first X!"); milestone unlocks.
- **Lifecycle**: account deactivation notice (when admin deactivates); data export ready (when user requests via sub-skill 03's `/account/data`).

### 3. Present the proposed list

Show the user a markdown table — one row per proposed trigger. Use the same columns:

| Trigger | When it fires | Why it's worth including for your product |
| --- | --- | --- |
| Signup confirmation | First successful signup | Confirms it worked + sets expectations |
| Low-usage warning | User has 20% of grant left | Lets users self-serve before they're blocked |
| (etc.) |

### 4. Ask for approval per row

For each row, the user can **approve**, **decline**, or **edit** (rename, change firing condition, change copy direction). Capture decisions before generating templates.

### 5. Generate templates autonomously

For approved triggers, write `lib/email-templates.ts` with one exported function per trigger (e.g., `signupConfirmationHtml(args)`, `lowUsageWarningHtml(args)`, `subscriptionReceiptHtml(args)`). Each template:

- Plain HTML, no external CSS or images, ≤ 200 lines.
- Inline styles only.
- Mobile-friendly (max-width: 600px container, full-width on small screens).
- Includes a footer line: *"You're receiving this because you have an account at &lt;product&gt;. &lt;unsubscribe-link&gt;"* — where the unsubscribe link is included only on **marketing-class** emails (digests, re-engagement, milestones), never on transactional ones (receipts, password reset, security alerts).

Skeleton:

```ts
// lib/email-templates.ts
const PRODUCT = process.env.PRODUCT_NAME ?? 'App';
const UNSUB = (email: string) => `${process.env.AUTH_URL}/account/unsubscribe?email=${encodeURIComponent(email)}`;

const shell = (bodyHtml: string, opts?: { marketing?: boolean; to?: string }) => `
<!doctype html>
<html><body style="margin:0;padding:0;background:#f6f6f6;font-family:-apple-system,Segoe UI,Roboto,sans-serif;color:#222;">
  <table role="presentation" width="100%" cellpadding="0" cellspacing="0" style="background:#f6f6f6;padding:24px 0;">
    <tr><td align="center">
      <table role="presentation" width="100%" cellpadding="0" cellspacing="0" style="max-width:600px;background:#fff;border-radius:8px;padding:32px;">
        <tr><td style="font-size:16px;line-height:1.5;">${bodyHtml}</td></tr>
        <tr><td style="padding-top:24px;font-size:12px;color:#888;border-top:1px solid #eee;margin-top:24px;">
          You're receiving this because you have an account at ${PRODUCT}.
          ${opts?.marketing && opts?.to ? ` <a href="${UNSUB(opts.to)}" style="color:#888;">Unsubscribe</a>.` : ''}
        </td></tr>
      </table>
    </td></tr>
  </table>
</body></html>`;

export function signupConfirmationHtml(args: { firstName: string }) {
  return shell(`<h1 style="margin:0 0 16px;font-size:22px;">Welcome, ${args.firstName}.</h1>
    <p>Your ${PRODUCT} account is ready. <a href="${process.env.AUTH_URL}">Sign in to get started</a>.</p>`);
}

export function lowUsageWarningHtml(args: { firstName: string; consumed: number; granted: number }) {
  const pct = Math.round((args.consumed / args.granted) * 100);
  return shell(`<h1 style="margin:0 0 16px;font-size:22px;">You've used ${pct}% of your quota</h1>
    <p>Hi ${args.firstName} — you have ${args.granted - args.consumed} of ${args.granted} actions left this period.</p>
    <p><a href="${process.env.AUTH_URL}/account">Manage your account</a>.</p>`);
}

// ...one function per approved trigger
```

### 6. Wire each template at its trigger point

For each approved trigger, add a call to `sendEmail(...)` at the right place in the codebase:

- **Signup confirmation** → end of the `signup` server action (after `signIn(...)`), `await sendEmail({ to: email, subject: 'Welcome to ...', html: signupConfirmationHtml({ firstName }) })`.
- **Email verification** → in the signup action *before* `signIn`, generate an `authTokens` row with `purpose: 'verify'` and email the link.
- **Password reset** → already wired in `/reset/request` route — replace the inline HTML with `passwordResetHtml({ url })`.
- **Sign-in from new device** → in the Auth.js `signIn` callback, compare request IP/UA against last-seen values stored on the user row; on mismatch send `newDeviceSignInHtml(...)`.
- **Waitlist confirmation** → end of the waitlist server action.
- **Approval invite** → in the admin "approve" action (sub-skill 07) right after inserting `allowedEmails`.
- **Low-usage warning** → inside `gatedAiCall` (sub-skill 07's helper) when `consumed/granted >= 0.8` AND `lastLowUsageWarningAt` (a new column on `users`) is more than 7 days ago. Update the column when sent.
- **Usage exhausted** → same helper, when the call is rejected because `consumed >= granted`.
- **Usage refilled** → in the admin "grant usage" action.
- **Subscription receipt / failed payment / cancelled / renewal reminder** → Stripe webhook handlers (`checkout.session.completed`, `invoice.payment_failed`, `customer.subscription.deleted`, scheduled job 3 days before `current_period_end`).
- **Weekly digest** → cron / scheduled job (sub-skill 09 if present).
- **Comment notification / subscriber confirmation** → in the comment-create / subscribe server actions.
- **Re-engagement / first-action / milestone** → background job comparing `users.lastSeenAt` and event counts.
- **Account deactivation notice** → admin "deactivate" action.
- **Data export ready** → background job that produces the export file, then emails the download link.

Add any new columns the wiring requires (e.g., `lastLowUsageWarningAt timestamp`, `lastSeenAt timestamp`, `lastSignInIp inet`, `lastSignInUa text`) to `lib/db/schema.ts` and run `drizzle-kit push`.

### 7. DEV/TEST helper — `lib/email-debug.ts`

In development, intercept `sendEmail` and write the rendered HTML to disk instead of actually sending — so the agent can show the user what each trigger looks like before going live:

```ts
// lib/email-debug.ts
import { mkdir, writeFile } from 'node:fs/promises';
import { join } from 'node:path';

export async function writeDebugEmail(args: { to: string; subject: string; html: string }) {
  const dir = join(process.cwd(), 'tmp', 'emails');
  await mkdir(dir, { recursive: true });
  const safe = args.subject.replace(/[^a-z0-9]+/gi, '-').slice(0, 60);
  const file = join(dir, `${Date.now()}-${safe}.html`);
  await writeFile(file, `<!-- to: ${args.to} | subject: ${args.subject} -->\n${args.html}`, 'utf8');
  return file;
}
```

Then short-circuit `sendEmail` in development:

```ts
// at the top of sendEmail() in lib/email.ts
if (process.env.NODE_ENV !== 'production' && process.env.EMAIL_DEBUG !== 'off') {
  const { writeDebugEmail } = await import('./email-debug');
  const path = await writeDebugEmail(args);
  console.log(`[email-debug] wrote ${path}`);
  return;
}
```

Add `tmp/` to `.gitignore`. After generating templates, the agent should trigger one of each via a small script or by walking the user through the corresponding flow, then `open tmp/emails/<file>.html` to preview.

## Auth.js — install and wire

```bash
npm install next-auth@beta @auth/drizzle-adapter   # or @auth/prisma-adapter
npm install bcryptjs                                # only if Credentials path
```

`auth.ts` (project root):
```ts
import NextAuth from 'next-auth';
import Google from 'next-auth/providers/google';
import Resend from 'next-auth/providers/resend';
import Credentials from 'next-auth/providers/credentials';
import bcrypt from 'bcryptjs';
import { DrizzleAdapter } from '@auth/drizzle-adapter';
import { db } from '@/lib/db';
import { users, allowedEmails } from '@/lib/db/schema';
import { eq } from 'drizzle-orm';

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: DrizzleAdapter(db),
  providers: [
    ...(process.env.AUTH_GOOGLE_ID ? [Google] : []),
    ...(process.env.RESEND_API_KEY ? [Resend({ from: process.env.EMAIL_FROM! })] : []),
    Credentials({
      credentials: { email: {}, password: {} },
      async authorize(creds) {
        if (!creds?.email || !creds?.password) return null;
        const [user] = await db.select().from(users).where(eq(users.email, String(creds.email)));
        if (!user || !user.passwordHash || user.deactivatedAt) return null;
        const ok = await bcrypt.compare(String(creds.password), user.passwordHash);
        return ok ? user : null;
      },
    }),
  ],
  callbacks: {
    // Free beta with waitlist (or paid): block sign-in unless the email is whitelisted.
    async signIn({ user }) {
      if (process.env.ACCESS_MODE === 'open') return true;
      if (!user.email) return false;
      const [allowed] = await db.select().from(allowedEmails).where(eq(allowedEmails.email, user.email));
      return Boolean(allowed);
    },
  },
  pages: { signIn: '/login' },
});
```

Wire the route handler `app/api/auth/[...nextauth]/route.ts`:
```ts
export { GET, POST } from '@/auth';
```

Server-side gate on protected routes:
```ts
const session = await auth();
if (!session?.user) redirect('/login');
```

## Database schema — users, waitlist, whitelist, consent, invites

Use Drizzle (or Prisma equivalent). The shape below is the minimum that supports every access mode and every admin-dashboard action in sub-skill 07.

```ts
// lib/db/schema.ts
import { pgTable, uuid, text, timestamp, integer, inet } from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  id: uuid('id').defaultRandom().primaryKey(),
  email: text('email').notNull().unique(),
  firstName: text('first_name').notNull(),
  lastName: text('last_name').notNull(),
  passwordHash: text('password_hash'),                    // null for OAuth-only users
  intendedUse: text('intended_use'),                      // optional "how do you plan to use this?"
  emailVerifiedAt: timestamp('email_verified_at'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  deactivatedAt: timestamp('deactivated_at'),             // set by admin to block login
  // Per-user usage budget (sub-skill 07 grants and decrements this).
  usageGranted: integer('usage_granted').notNull().default(0),
  usageConsumed: integer('usage_consumed').notNull().default(0),
});

// Public waitlist signups before admin approval.
export const waitlist = pgTable('waitlist', {
  id: uuid('id').defaultRandom().primaryKey(),
  email: text('email').notNull().unique(),
  intendedUse: text('intended_use'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  approvedAt: timestamp('approved_at'),
  rejectedAt: timestamp('rejected_at'),
});

// Emails the admin has whitelisted for self-signup. Approving from the waitlist
// inserts here. Adding a user via the admin dashboard also inserts here.
export const allowedEmails = pgTable('allowed_emails', {
  email: text('email').primaryKey(),
  invitedBy: uuid('invited_by'),                          // admin user id (or null for system)
  invitedAt: timestamp('invited_at').defaultNow().notNull(),
  inviteTokenHash: text('invite_token_hash'),             // null when admin chose "whitelist only" (no email sent)
  inviteAcceptedAt: timestamp('invite_accepted_at'),
  expiresAt: timestamp('expires_at'),                     // token expiry (typically +14d)
});

// Versioned consent records: who agreed to what, when.
export const userConsents = pgTable('user_consents', {
  id: uuid('id').defaultRandom().primaryKey(),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  consentType: text('consent_type').notNull(),            // 'tos' | 'privacy' | 'marketing'
  documentVersion: text('document_version').notNull(),    // e.g., '2026-04-19' (last_updated of the doc)
  acceptedAt: timestamp('accepted_at').defaultNow().notNull(),
  ipAddress: inet('ip_address'),
});

// Password-reset and email-verification tokens (single table, scoped by purpose).
export const authTokens = pgTable('auth_tokens', {
  tokenHash: text('token_hash').primaryKey(),             // store sha256(token); never the raw token
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  purpose: text('purpose').notNull(),                     // 'verify' | 'reset'
  expiresAt: timestamp('expires_at').notNull(),
  consumedAt: timestamp('consumed_at'),
});
```

Run the migration before wiring the form (`drizzle-kit push` for MVP; proper migrations later).

## Signup flow — the form everyone agreed on

The signup form has the **same fields regardless of access mode**. What differs is whether the form even renders, and what happens after submit.

Required fields:
- **Email** (`type="email" required autocomplete="email"`)
- **First name** (`required autocomplete="given-name"`)
- **Last name** (`required autocomplete="family-name"`)
- **Password** (`type="password" required minlength="12" autocomplete="new-password"`) — only on Credentials path
- **Optional**: "How do you plan to use this?" textarea (rows=3, maxlength=500)
- **Required checkbox**: *"I agree to the [Terms of Service](/terms) and [Privacy Policy](/privacy)."* (cannot submit without it)
- **Optional checkbox** (unticked default): *"I'd like occasional product updates by email."*

Submit handler must, in order:
1. Re-check email-allowed status against `allowedEmails` if `ACCESS_MODE !== 'open'`.
2. Hash password with `bcryptjs` (cost 10–12), store as `passwordHash`.
3. Insert `users` row with the initial `usageGranted` from `INITIAL_USAGE_GRANT` env var (set in sub-skill 07).
4. Insert `userConsents` rows: one for `tos`, one for `privacy`, plus one for `marketing` if checked. Each row stores the document `last_updated` date as `documentVersion`, plus the request IP.
5. (If invite token in URL) mark `allowedEmails.inviteAcceptedAt`.
6. Sign the user in immediately (or send verification email first if you require pre-verification).

```tsx
// app/signup/page.tsx
import { headers } from 'next/headers';
import { db } from '@/lib/db';
import { users, allowedEmails, userConsents } from '@/lib/db/schema';
import { eq } from 'drizzle-orm';
import bcrypt from 'bcryptjs';
import { signIn } from '@/auth';

const TOS_VERSION = '2026-04-19';      // bump when the doc changes
const PRIVACY_VERSION = '2026-04-19';

export default function SignupPage({ searchParams }: { searchParams: { token?: string; email?: string } }) {
  return (
    <form action={async (formData: FormData) => {
      'use server';
      const email = String(formData.get('email')).toLowerCase().trim();
      const firstName = String(formData.get('firstName')).trim();
      const lastName = String(formData.get('lastName')).trim();
      const password = String(formData.get('password'));
      const intendedUse = String(formData.get('intendedUse') ?? '').trim() || null;
      const acceptTos = formData.get('acceptTos') === 'on';
      const acceptMarketing = formData.get('acceptMarketing') === 'on';

      if (!acceptTos) throw new Error('You must agree to the Terms and Privacy Policy.');

      if (process.env.ACCESS_MODE !== 'open') {
        const [allowed] = await db.select().from(allowedEmails).where(eq(allowedEmails.email, email));
        if (!allowed) throw new Error('This email is not on the access list. Join the waitlist to request access.');
      }

      const passwordHash = await bcrypt.hash(password, 12);
      const initialGrant = Number(process.env.INITIAL_USAGE_GRANT ?? 0);
      const [user] = await db.insert(users).values({
        email, firstName, lastName, passwordHash, intendedUse, usageGranted: initialGrant,
      }).returning();

      const ip = (await headers()).get('x-forwarded-for')?.split(',')[0] ?? null;
      await db.insert(userConsents).values([
        { userId: user.id, consentType: 'tos',     documentVersion: TOS_VERSION,     ipAddress: ip },
        { userId: user.id, consentType: 'privacy', documentVersion: PRIVACY_VERSION, ipAddress: ip },
        ...(acceptMarketing ? [{ userId: user.id, consentType: 'marketing', documentVersion: PRIVACY_VERSION, ipAddress: ip }] : []),
      ]);

      if (searchParams.token) {
        await db.update(allowedEmails).set({ inviteAcceptedAt: new Date() }).where(eq(allowedEmails.email, email));
      }

      await signIn('credentials', { email, password, redirectTo: '/' });
    }} className="max-w-md mx-auto space-y-4 p-6">
      <h1 className="text-3xl font-semibold">Create your account</h1>

      <input name="email"     type="email"    required defaultValue={searchParams.email ?? ''} className="input input-bordered w-full" placeholder="Email" autoComplete="email" />
      <input name="firstName" type="text"     required className="input input-bordered w-full" placeholder="First name" autoComplete="given-name" />
      <input name="lastName"  type="text"     required className="input input-bordered w-full" placeholder="Last name"  autoComplete="family-name" />
      <input name="password"  type="password" required minLength={12} className="input input-bordered w-full" placeholder="Password (min 12 chars)" autoComplete="new-password" />

      <textarea name="intendedUse" rows={3} maxLength={500} className="textarea textarea-bordered w-full" placeholder="How do you plan to use this? (optional)" />

      <label className="label cursor-pointer justify-start gap-2">
        <input type="checkbox" name="acceptTos" required className="checkbox checkbox-sm" />
        <span className="label-text">I agree to the <a className="link" href="/terms">Terms of Service</a> and <a className="link" href="/privacy">Privacy Policy</a>.</span>
      </label>
      <label className="label cursor-pointer justify-start gap-2">
        <input type="checkbox" name="acceptMarketing" className="checkbox checkbox-sm" />
        <span className="label-text">I'd like occasional product updates by email. (Optional, unsubscribe any time.)</span>
      </label>

      <button className="btn btn-primary w-full">Create account</button>
    </form>
  );
}
```

If `ACCESS_MODE` is `waitlist` or `paid`, the public `/signup` page should redirect to `/waitlist` unless an `?email=&token=` invite query is present.

## Waitlist mode (only if `ACCESS_MODE === 'waitlist'` or `'paid'`)

Public `/waitlist` page:

```tsx
// app/waitlist/page.tsx
import { db } from '@/lib/db';
import { waitlist } from '@/lib/db/schema';

export default function WaitlistPage() {
  return (
    <form action={async (formData: FormData) => {
      'use server';
      const email = String(formData.get('email')).toLowerCase().trim();
      const intendedUse = String(formData.get('intendedUse') ?? '').trim() || null;
      await db.insert(waitlist).values({ email, intendedUse }).onConflictDoNothing();
    }} className="max-w-md mx-auto space-y-4 p-6">
      <h1 className="text-3xl font-semibold">Join the waitlist</h1>
      <p className="opacity-75">We're letting people in gradually. Drop your email and we'll be in touch.</p>
      <input name="email" type="email" required className="input input-bordered w-full" placeholder="Email" />
      <textarea name="intendedUse" rows={3} maxLength={500} className="textarea textarea-bordered w-full" placeholder="How do you plan to use this? (optional)" />
      <button className="btn btn-primary w-full">Request access</button>
    </form>
  );
}
```

When the admin (sub-skill 07) approves a waitlist row, two paths:

- **Email service available** — generate a one-time invite token, store `sha256(token)` as `inviteTokenHash`, send an email with link `https://<domain>/signup?email=<email>&token=<token>`. User clicks, signup form prefills email, and on submit the token is validated and consumed.
- **Email service not available** — just insert into `allowedEmails` with no token. The user is told (out-of-band, e.g., the admin emails them personally) to go to `/signup` and create their account. The Auth.js `signIn` callback only allows whitelisted emails through, so unauthorized users can't slip in.

The admin dashboard (sub-skill 07) detects which mode using `emailEnabled` from `lib/email.ts`.

## Password reset (Credentials path)

Two routes: request, and confirm.

```ts
// app/(auth)/reset/request/route.ts — POST { email }
import crypto from 'node:crypto';
import { db } from '@/lib/db';
import { users, authTokens } from '@/lib/db/schema';
import { eq } from 'drizzle-orm';
import { sendEmail, emailEnabled } from '@/lib/email';

export async function POST(req: Request) {
  if (!emailEnabled) return new Response('Email not configured', { status: 503 });
  const { email } = await req.json();
  const [user] = await db.select().from(users).where(eq(users.email, email));
  // Always respond 200 to avoid user-enumeration. Only send if the user exists.
  if (user) {
    const raw = crypto.randomBytes(32).toString('base64url');
    const tokenHash = crypto.createHash('sha256').update(raw).digest('hex');
    const expiresAt = new Date(Date.now() + 60 * 60 * 1000); // 1h
    await db.insert(authTokens).values({ tokenHash, userId: user.id, purpose: 'reset', expiresAt });
    const url = `${process.env.AUTH_URL}/reset/confirm?token=${raw}`;
    await sendEmail({
      to: email,
      subject: 'Reset your password',
      html: `<p>Click to reset your password (expires in 1 hour):</p><p><a href="${url}">${url}</a></p>`,
    });
  }
  return new Response('ok');
}
```

```ts
// app/(auth)/reset/confirm/route.ts — POST { token, newPassword }
import crypto from 'node:crypto';
import bcrypt from 'bcryptjs';
import { db } from '@/lib/db';
import { users, authTokens } from '@/lib/db/schema';
import { eq, and, isNull, gt } from 'drizzle-orm';

export async function POST(req: Request) {
  const { token, newPassword } = await req.json();
  if (newPassword.length < 12) return new Response('Password too short', { status: 400 });
  const tokenHash = crypto.createHash('sha256').update(token).digest('hex');
  const [row] = await db.select().from(authTokens).where(
    and(eq(authTokens.tokenHash, tokenHash), eq(authTokens.purpose, 'reset'), isNull(authTokens.consumedAt), gt(authTokens.expiresAt, new Date()))
  );
  if (!row) return new Response('Invalid or expired token', { status: 400 });
  await db.update(users).set({ passwordHash: await bcrypt.hash(newPassword, 12) }).where(eq(users.id, row.userId));
  await db.update(authTokens).set({ consumedAt: new Date() }).where(eq(authTokens.tokenHash, tokenHash));
  return new Response('ok');
}
```

Build minimal pages at `/reset/request` (form: email) and `/reset/confirm` (form: new password, token from URL).

Email-verification at signup uses the same `authTokens` table with `purpose: 'verify'`. Optional for MVP — recommend it for any project that ships marketing emails (otherwise spam traps poison your sender reputation).

## OAuth providers — console walkthroughs

If the user picked Google or GitHub, walk them through the console step-by-step. **Offer to drive a browser** (see SKILL.md "Browser automation"): you can `open https://...` to take them to the right page, or run a Playwright script that opens a window and waits at credential-entry steps.

### Google (Google Cloud Console)
1. Go to https://console.cloud.google.com/apis/credentials.
2. Create a project if needed. Top-left dropdown &rarr; **New Project**.
3. **OAuth consent screen** &rarr; External &rarr; fill app name, user support email, developer contact &rarr; Save.
4. **Credentials** &rarr; **Create Credentials** &rarr; **OAuth client ID** &rarr; Web application.
5. Authorized redirect URI: `http://localhost:3000/api/auth/callback/google` (and the production URL after deploy).
6. Copy Client ID + Client Secret. Paste them.

### GitHub
1. Go to https://github.com/settings/developers &rarr; **New OAuth App**.
2. Name, homepage URL, callback URL: `http://localhost:3000/api/auth/callback/github`.
3. Generate a client secret. Copy ID + secret. Paste them.

For OAuth users, the first sign-in inserts a `users` row with no `passwordHash`. Capture first/last name from the OAuth profile (Google provides `given_name` / `family_name`); ask for `intendedUse` and consent on a brief one-screen onboarding immediately after first sign-in if `users.intendedUse` is null and `userConsents` is empty.

## Anti-patterns to avoid

- Building custom auth UI before Auth.js is wired. Get the flow working with stock components first; restyle later.
- Sending email from your own SMTP server. Use Resend (or Postmark) — it's free for MVP volume and handles deliverability.
- Calling `auth()` inside React Server Components multiple times for the same request. Wrap once at the top of the page.
- Storing the password hash in plaintext columns "for now." There's no "for now" with passwords.
- **Letting waitlist signups skip the consent checkboxes.** The waitlist form is a public marketing form, not an account; it doesn't need TOS/Privacy consent. The *signup* form (after approval) is where consent is captured. Don't blur the two.
- **Pre-ticked marketing consent.** Void under GDPR. Always unticked default.
- **Storing raw invite/reset tokens in the DB.** Always store `sha256(token)` and discard the raw value after sending it.
- **Returning a different status from `/reset/request` based on whether the email exists.** That's user enumeration. Always respond `200 ok`.

## Exit criteria

- The user can sign up, sign in, and sign out on `localhost`.
- The signup form requires first name, last name, email, password (Credentials path), the TOS+Privacy checkbox, and offers an unticked marketing checkbox plus an optional intended-use textarea.
- `userConsents` has a row for every consent each user gave, including the document version and request IP.
- If the access model is `waitlist` or `paid`, the public `/signup` page rejects (or 404s) and `/waitlist` is the only public entry.
- Password reset works end-to-end on `localhost` (request → email → click link → set new password → log in).
- `lib/email.ts` exports `emailEnabled` so sub-skill 07 can branch invite-vs-whitelist.
- `.env.local` contains every secret used and is gitignored.
- A line in `PROJECT.md` under `# Decisions` records the chosen auth path, the access mode, and the document versions referenced by signup (`TOS_VERSION`, `PRIVACY_VERSION`).
- If email is configured, every approved trigger has a template in `lib/email-templates.ts` and a wired call site.
- The DEV email-debug helper writes to `tmp/emails/` and is gitignored.

Move on to `05-ai-integration.md`.
