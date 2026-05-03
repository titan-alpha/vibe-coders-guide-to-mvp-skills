# 07 · Admin Dashboard

Goal: add a private `/admin` route where the project owner can (a) see KPIs that actually matter for *their* MVP and (b) operate the user base — add users, approve from the waitlist, grant or limit usage, deactivate. Built bottom-up from the codebase, not top-down from a generic template.

## DIALOGUE — does the project want one?

For any project with auth (any access mode from sub-skill 01), the dashboard is **strongly recommended** — without it the founder has no way to add users, see who's waiting, or grant usage.

For a static content site with no auth, ask plainly:

> *"Want a private admin dashboard at `/admin` where you can see KPIs and metrics for your MVP? It would be locked to a password you pick."*

If the project has auth, frame it differently:

> *"You'll need an admin dashboard to manage the people on the platform — approve waitlist signups, add users by email, grant usage, deactivate misbehaving accounts. I'll build that plus a few KPIs tailored to your MVP. Sound good?"*

Then, in the same exchange, ask about notifications:

> *"The dashboard will also let you send notifications to all users or to specific users — they appear in a bell icon at the top-right of the site. OK to include that, or skip notifications and just have the user-management tabs?"*

If the user opts out of notifications: skip Tab 5 entirely AND tell them to inform sub-skill 02 to leave out the bell icon (or note in `STATE.md` `# Decisions`: "Notification center: skipped — header has no bell icon").

If the user declines the dashboard altogether, skip and move on to `08-monetization.md`. (For a project with auth, gently flag that without an admin dashboard they'll be running SQL by hand to manage users — they'll likely come back to this.)

## AUTONOMOUS — analyze the codebase

Build a mental model of what data already flows through the project.

### 1. Inventory existing data

- **Database schema.** Read every migration / model definition. List the tables and the lifecycle of each row: created when? updated when? what triggers each change?
- **User events.** Grep for any analytics calls (PostHog, Mixpanel, plain `console.log`, etc.) and any `console`/log statements that imply tracked behavior. Note what's already captured.
- **Auth.** Read sub-skill 04's schema. You should see `users`, `waitlist`, `allowedEmails`, `userConsents`, `authTokens`. The `users` table has `usageGranted` and `usageConsumed` columns to power the Usage tab.
- **Money flow.** Any Stripe / payment integration? What events fire? (Sub-skill 08 may or may not be done yet — if not, leave hooks for later.)
- **AI usage.** Any calls through `lib/ai.ts`? If yes, that's where you'll log per-user usage (see "Usage instrumentation" below).

Write the inventory to a scratch note (mentally or in a temp file). Don't show this to the user yet — it's input to step 3.

### 2. Research KPIs for this kind of project

Read `PROJECT.md` for the audience and MVP slice. Then choose the 4–6 KPIs that actually tell the founder if the MVP is working. Suggested starting points by project type:

| Project type | Core KPIs |
| --- | --- |
| Two-sided marketplace | Supply count, demand count, time-to-match, take rate, weekly retention of repeat users |
| Content product | DAU, sessions, depth-per-session, return rate, top 10 content items |
| SaaS tool | Signups, activation rate (define the "aha" event), weekly retention, conversion to paid |
| AI-feature product | Requests per user, success rate (parsed without error), p95 latency, cost per user, top-used feature |
| Social / community | DAU, posts per active user, comment / reply ratio, retention by cohort week |

Don't pad with vanity metrics. Five charts that change behavior beat fifty that don't.

### 3. Identify the gap

For each chosen KPI, ask: *is this data already being captured?*

- **Already captured** → great, query it.
- **Capturable with a small change** → propose a minimal change. Examples: "add a `signed_up_at` timestamp," "log a `feature_used` event in `lib/ai.ts`," "record `referrer` on the signup form."
- **Requires major instrumentation** → defer. Note as post-MVP.

## DIALOGUE — propose KPIs and the initial usage budget

Come back to the user with a concrete proposal:

> *"Here are the 5 KPIs I'd track for your MVP:*
> 1. *DAU — do people come back?*
> 2. *Activation rate — do new users hit the aha moment?*
> 3. *Top 5 used features — where is value?*
> 4. *AI cost per user — is the unit economics sane?*
> 5. *7-day retention by signup cohort — are we keeping people?*
>
> *Of those, 3 we can compute today from existing data. 2 need small additions:*
> - *We'd add a `signed_up_at` column to `users` (one migration).*
> - *We'd log a `feature_used` event in `lib/ai.ts` (a few lines).*"

Then, in the same conversation, propose the **initial usage grant per user** — the number that goes into `INITIAL_USAGE_GRANT` and gets baked into every new `users.usageGranted` value at signup.

The right number depends on the product. Reason out loud, then propose:

> *"Each AI call through your `lib/ai.ts` costs roughly $0.0002 (gpt-5-nano + minimal). If we want a beta user to be able to do ~100 meaningful actions before you'd want to top them up, that's about $0.02 per user — comfortable. I'd suggest **`INITIAL_USAGE_GRANT=100`**. From the dashboard you'll be able to grant more — to one user, or to all users in bulk. Want to start at 100 or pick a different number?"*

Heuristics, by product type:

| Product | Suggested initial grant | Reasoning |
| --- | --- | --- |
| AI-feature SaaS (one call per action) | 50–200 | Enough for a real evaluation, low enough to detect runaway use |
| Content product with 1 AI summary per page | 1000+ | Reads cost almost nothing; cap is more about abuse than economics |
| Image generation / heavy AI | 5–20 | Each call costs cents, not millicents |
| Marketplace / no AI | irrelevant — set to a high number and ignore | Usage gating doesn't apply |

Iterate until both the KPI list and the usage number feel right. Persist `INITIAL_USAGE_GRANT` in `.env.local`.

## AUTONOMOUS — usage instrumentation

If the product has any per-user usage that needs gating (AI calls, generations, exports, anything that costs you money or compute per call), wire it through a tiny helper. Example for AI:

```ts
// lib/ai-gated.ts
import 'server-only';
import { db } from '@/lib/db';
import { users } from '@/lib/db/schema';
import { eq, sql } from 'drizzle-orm';
import { aiCall } from '@/lib/ai';

export class UsageExhausted extends Error {}

export async function gatedAiCall<T>(args: { userId: string } & Parameters<typeof aiCall>[0]) {
  const [u] = await db.select({
    granted: users.usageGranted, consumed: users.usageConsumed, deactivated: users.deactivatedAt,
  }).from(users).where(eq(users.id, args.userId));
  if (!u || u.deactivated) throw new UsageExhausted('Account deactivated');
  if (u.consumed >= u.granted) throw new UsageExhausted('Usage exhausted — contact admin');

  const result = await aiCall(args);
  await db.update(users).set({ usageConsumed: sql`${users.usageConsumed} + 1` }).where(eq(users.id, args.userId));
  return result;
}
```

Replace `aiCall` callers with `gatedAiCall` everywhere a user triggered the call. (Background jobs and admin-triggered calls should use raw `aiCall`.) Surface `UsageExhausted` to the UI as a friendly "you've used your monthly quota — contact us" message.

## AUTONOMOUS — lock the route with a password (placeholder + forced change on first login)

The admin dashboard is for the **founder only**, not regular users. Don't tie it to the user-account system — gate it with a dedicated password the founder picks. This works regardless of which auth mode the rest of the app uses.

The flow has three states:

1. **Setup** — agent generates a placeholder password, shows it to the user once, and stores it in `.env.local` as `ADMIN_PASSWORD_BOOTSTRAP`.
2. **First login** — user signs in with the placeholder. Middleware accepts it but redirects to `/admin/change-password`. User cannot reach any admin page until they set a real password.
3. **Steady state** — middleware checks the bcrypt hash stored in the DB. The bootstrap env var is no longer consulted (and the agent tells the user they can delete it).

### Step 1 — generate the placeholder + show it to the user

Use a memorable-but-random pattern so it's easy to type once:

```bash
node -e "console.log('vibe-admin-' + crypto.randomBytes(4).toString('hex'))"
# example output: vibe-admin-9f3a2c8b
```

Append to `.env.local`:
```
ADMIN_PASSWORD_BOOTSTRAP=vibe-admin-9f3a2c8b   # placeholder; remove after first login
INITIAL_USAGE_GRANT=100                          # baked into every new user's usageGranted at signup
```

Tell the user, in chat, in plain English:

> *"Your one-time admin password is **`vibe-admin-9f3a2c8b`**.*
>
> *Open `/admin` (or `https://<your-domain>/admin` once deployed). Sign in with username `admin` and that password. The first thing you'll see is a 'Set your admin password' form — pick something you'll remember, at least 12 characters. After that, the placeholder stops working and you can delete `ADMIN_PASSWORD_BOOTSTRAP` from `.env.local` (and from Vercel env).*"

### Step 2 — schema for the persistent admin password

Add a single-row table (or a one-row settings table if you prefer one bag for all admin settings):

```ts
// lib/db/schema.ts (extend)
export const adminSettings = pgTable('admin_settings', {
  id: integer('id').primaryKey().default(1),         // single-row pattern
  passwordHash: text('password_hash'),               // null until first change
  passwordSetAt: timestamp('password_set_at'),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
});
```

Seed the row at migration time:
```sql
insert into admin_settings (id) values (1) on conflict do nothing;
```

### Step 3 — middleware checks hash first, then bootstrap

```ts
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export async function middleware(req: NextRequest) {
  if (!req.nextUrl.pathname.startsWith('/admin')) return NextResponse.next();

  // /admin/change-password is the one admin route reachable with bootstrap creds.
  // Allow it through with bootstrap auth so the user can land here and set the real password.
  const isChangePage = req.nextUrl.pathname === '/admin/change-password';

  const authHeader = req.headers.get('authorization');
  if (!authHeader?.startsWith('Basic ')) {
    return new NextResponse('Authentication required', {
      status: 401,
      headers: { 'WWW-Authenticate': 'Basic realm="Admin"' },
    });
  }
  const [user, pass] = Buffer.from(authHeader.slice(6), 'base64').toString().split(':');
  if (user !== 'admin' || !pass) return new NextResponse('Auth failed', { status: 401 });

  // Edge runtime can't import bcryptjs cleanly; do the check via a lightweight API route.
  const result = await fetch(`${req.nextUrl.origin}/api/admin/verify`, {
    method: 'POST',
    body: JSON.stringify({ password: pass }),
  }).then(r => r.json() as Promise<{ ok: boolean; mustChange: boolean }>);

  if (!result.ok) {
    return new NextResponse('Auth failed', {
      status: 401,
      headers: { 'WWW-Authenticate': 'Basic realm="Admin"' },
    });
  }

  if (result.mustChange && !isChangePage) {
    return NextResponse.redirect(new URL('/admin/change-password', req.url));
  }

  return NextResponse.next();
}

export const config = { matcher: '/admin/:path*' };
```

```ts
// app/api/admin/verify/route.ts
import bcrypt from 'bcryptjs';
import { db } from '@/lib/db';
import { adminSettings } from '@/lib/db/schema';
import { eq } from 'drizzle-orm';

export async function POST(req: Request) {
  const { password } = (await req.json()) as { password: string };
  const [row] = await db.select().from(adminSettings).where(eq(adminSettings.id, 1));

  // Persistent hash exists → use it. Bootstrap env var is no longer consulted.
  if (row?.passwordHash) {
    const ok = await bcrypt.compare(password, row.passwordHash);
    return Response.json({ ok, mustChange: false });
  }

  // No persistent hash yet → accept bootstrap and force a change on first page hit.
  const bootstrap = process.env.ADMIN_PASSWORD_BOOTSTRAP;
  if (bootstrap && password === bootstrap) {
    return Response.json({ ok: true, mustChange: true });
  }

  return Response.json({ ok: false, mustChange: false });
}
```

### Step 4 — the change-password page

```tsx
// app/admin/change-password/page.tsx
import bcrypt from 'bcryptjs';
import { db } from '@/lib/db';
import { adminSettings } from '@/lib/db/schema';
import { eq } from 'drizzle-orm';
import { redirect } from 'next/navigation';

export default function ChangePasswordPage() {
  return (
    <main className="max-w-md mx-auto p-6">
      <h1 className="text-2xl font-semibold">Set your admin password</h1>
      <p className="text-sm opacity-75 mt-2">
        You're using the one-time placeholder password. Pick a real one (≥ 12 characters).
        After this, the placeholder stops working — you can remove <code>ADMIN_PASSWORD_BOOTSTRAP</code> from your env.
      </p>
      <form
        className="mt-6 space-y-3"
        action={async (formData: FormData) => {
          'use server';
          const next = String(formData.get('newPassword'));
          const confirm = String(formData.get('confirmPassword'));
          if (next.length < 12) throw new Error('Password must be at least 12 characters.');
          if (next !== confirm) throw new Error('Passwords do not match.');
          const passwordHash = await bcrypt.hash(next, 12);
          await db.update(adminSettings)
            .set({ passwordHash, passwordSetAt: new Date(), updatedAt: new Date() })
            .where(eq(adminSettings.id, 1));
          redirect('/admin');
        }}
      >
        <input name="newPassword"     type="password" required minLength={12} className="input input-bordered w-full" placeholder="New password (min 12)" autoComplete="new-password" />
        <input name="confirmPassword" type="password" required minLength={12} className="input input-bordered w-full" placeholder="Confirm new password" autoComplete="new-password" />
        <button className="btn btn-primary w-full">Set password</button>
      </form>
    </main>
  );
}
```

After the user submits, the persistent hash is set, the next request validates against it, `mustChange` becomes false, and `/admin` renders normally. The browser's stored Basic Auth credentials no longer match the bootstrap, so the user will be re-prompted for their new password — that's the one they typed in the form.

### Step 5 — tell the user to delete the bootstrap env var

Once the change-password flow has succeeded, surface a one-line note in the chat:

> *"Done. You can now remove `ADMIN_PASSWORD_BOOTSTRAP` from `.env.local` and Vercel env — it's no longer used. Your real password lives only in the database (as a bcrypt hash)."*

For better UX (custom login form instead of the browser prompt), build `/admin/login` that POSTs the password to a Route Handler, which sets a signed `admin_session` cookie on success. Middleware then checks the cookie. This is a v1.5 upgrade — basic auth + the change-password flow are enough to ship.

## AUTONOMOUS — build the dashboard with five tabs

`/admin` has five tabs: **Overview**, **Users**, **Waitlist**, **Usage**, **Notifications**. (Skip Notifications if the user opted out in the DIALOGUE above — see `STATE.md # Decisions`.) Use DaisyUI tabs (`tabs tabs-bordered`) or shadcn-style segmented routes — either works.

```tsx
// app/admin/layout.tsx
import Link from 'next/link';

export default function AdminLayout({ children }: { children: React.ReactNode }) {
  return (
    <main className="max-w-6xl mx-auto p-6 space-y-6">
      <header className="flex items-baseline justify-between">
        <h1 className="text-2xl font-semibold">Admin</h1>
      </header>
      <nav className="tabs tabs-bordered">
        <Link href="/admin"               className="tab">Overview</Link>
        <Link href="/admin/users"         className="tab">Users</Link>
        <Link href="/admin/waitlist"      className="tab">Waitlist</Link>
        <Link href="/admin/usage"         className="tab">Usage</Link>
        <Link href="/admin/notifications" className="tab">Notifications</Link>
      </nav>
      {children}
    </main>
  );
}
```

### Tab 1 — Overview (KPIs)

The original KPI dashboard. Use `unstable_cache` with a 60s revalidate.

```tsx
// app/admin/page.tsx
import { getKpis } from '@/lib/admin-kpis';
import { KpiCards, RetentionChart, TopFeatures } from './components';

export const revalidate = 60;

export default async function AdminOverview() {
  const kpis = await getKpis();
  return (
    <>
      <KpiCards data={kpis.summary} />
      <div className="grid lg:grid-cols-2 gap-4">
        <RetentionChart data={kpis.retention} />
        <TopFeatures data={kpis.topFeatures} />
      </div>
    </>
  );
}
```

Charts: install **recharts** (`npm install recharts`).

### Tab 2 — Users

List every user. Per-row actions: deactivate / reactivate, grant +N usage, view details.

```tsx
// app/admin/users/page.tsx
import { db } from '@/lib/db';
import { users } from '@/lib/db/schema';
import { desc } from 'drizzle-orm';
import { addUserAction, grantUsageAction, deactivateUserAction } from './actions';
import { emailEnabled } from '@/lib/email';

export default async function UsersTab() {
  const rows = await db.select().from(users).orderBy(desc(users.createdAt)).limit(500);
  return (
    <section className="space-y-6">
      <form action={addUserAction} className="flex gap-2 items-end flex-wrap">
        <label className="form-control flex-1 min-w-[260px]">
          <span className="label-text">Add user by email</span>
          <input name="email" type="email" required className="input input-bordered" placeholder="user@example.com" />
        </label>
        <button className="btn btn-primary" type="submit">
          {emailEnabled ? 'Send invite' : 'Whitelist email'}
        </button>
        <p className="text-xs opacity-70 basis-full">
          {emailEnabled
            ? 'A magic-link invite will be emailed. They click, complete signup, and they\'re in.'
            : 'No email service configured. The user will be allowed to sign up at /signup with a password they choose. Tell them out-of-band.'}
        </p>
      </form>

      <table className="table">
        <thead><tr>
          <th>Email</th><th>Name</th><th>Joined</th><th>Status</th><th>Usage</th><th>Actions</th>
        </tr></thead>
        <tbody>
          {rows.map(u => (
            <tr key={u.id} className={u.deactivatedAt ? 'opacity-50' : ''}>
              <td>{u.email}</td>
              <td>{u.firstName} {u.lastName}</td>
              <td>{u.createdAt.toISOString().slice(0, 10)}</td>
              <td>{u.deactivatedAt ? 'Deactivated' : 'Active'}</td>
              <td>{u.usageConsumed} / {u.usageGranted}</td>
              <td className="space-x-2">
                <form action={grantUsageAction} className="inline">
                  <input type="hidden" name="userId" value={u.id} />
                  <input type="number" name="amount" defaultValue={100} className="input input-xs input-bordered w-16" />
                  <button className="btn btn-xs">Grant</button>
                </form>
                <form action={deactivateUserAction} className="inline">
                  <input type="hidden" name="userId" value={u.id} />
                  <input type="hidden" name="action" value={u.deactivatedAt ? 'reactivate' : 'deactivate'} />
                  <button className="btn btn-xs btn-error">{u.deactivatedAt ? 'Reactivate' : 'Deactivate'}</button>
                </form>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </section>
  );
}
```

`actions.ts`:

```ts
'use server';
import crypto from 'node:crypto';
import { db } from '@/lib/db';
import { users, allowedEmails } from '@/lib/db/schema';
import { eq, sql } from 'drizzle-orm';
import { sendEmail, emailEnabled } from '@/lib/email';

export async function addUserAction(formData: FormData) {
  const email = String(formData.get('email')).toLowerCase().trim();
  if (emailEnabled) {
    const raw = crypto.randomBytes(32).toString('base64url');
    const hash = crypto.createHash('sha256').update(raw).digest('hex');
    const expiresAt = new Date(Date.now() + 14 * 24 * 60 * 60 * 1000);
    await db.insert(allowedEmails).values({ email, inviteTokenHash: hash, expiresAt }).onConflictDoUpdate({
      target: allowedEmails.email,
      set: { inviteTokenHash: hash, expiresAt, inviteAcceptedAt: null },
    });
    const url = `${process.env.AUTH_URL}/signup?email=${encodeURIComponent(email)}&token=${raw}`;
    await sendEmail({
      to: email,
      subject: "You're invited",
      html: `<p>You have access. Set up your account here (link expires in 14 days):</p><p><a href="${url}">${url}</a></p>`,
    });
  } else {
    await db.insert(allowedEmails).values({ email }).onConflictDoNothing();
  }
}

export async function grantUsageAction(formData: FormData) {
  const userId = String(formData.get('userId'));
  const amount = Number(formData.get('amount'));
  if (!Number.isFinite(amount) || amount <= 0) return;
  await db.update(users).set({ usageGranted: sql`${users.usageGranted} + ${amount}` }).where(eq(users.id, userId));
}

export async function deactivateUserAction(formData: FormData) {
  const userId = String(formData.get('userId'));
  const action = String(formData.get('action'));
  await db.update(users).set({
    deactivatedAt: action === 'deactivate' ? new Date() : null,
  }).where(eq(users.id, userId));
}
```

Add a global "Grant +N to all active users" form at the top of the tab (one-line server action that updates every non-deactivated user).

### Tab 3 — Waitlist

List every pending waitlist entry. Per-row actions: approve (which calls `addUserAction` under the hood, then marks the waitlist row `approvedAt`), reject.

```tsx
// app/admin/waitlist/page.tsx
import { db } from '@/lib/db';
import { waitlist } from '@/lib/db/schema';
import { isNull, desc } from 'drizzle-orm';
import { approveWaitlistAction, rejectWaitlistAction } from './actions';
import { emailEnabled } from '@/lib/email';

export default async function WaitlistTab() {
  const pending = await db.select().from(waitlist)
    .where(isNull(waitlist.approvedAt))
    .orderBy(desc(waitlist.createdAt));
  return (
    <section className="space-y-4">
      <p className="opacity-75 text-sm">
        Approving {emailEnabled ? 'sends a magic-link invite email.' : 'whitelists the email — tell the user to sign up at /signup with a password they choose.'}
      </p>
      <table className="table">
        <thead><tr><th>Email</th><th>Joined</th><th>Intended use</th><th>Actions</th></tr></thead>
        <tbody>
          {pending.map(w => (
            <tr key={w.id}>
              <td>{w.email}</td>
              <td>{w.createdAt.toISOString().slice(0, 10)}</td>
              <td className="max-w-md truncate">{w.intendedUse ?? '—'}</td>
              <td className="space-x-2">
                <form action={approveWaitlistAction} className="inline">
                  <input type="hidden" name="id" value={w.id} />
                  <button className="btn btn-xs btn-success">Approve</button>
                </form>
                <form action={rejectWaitlistAction} className="inline">
                  <input type="hidden" name="id" value={w.id} />
                  <button className="btn btn-xs btn-ghost">Reject</button>
                </form>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </section>
  );
}
```

`actions.ts` for the waitlist tab:

```ts
'use server';
import { db } from '@/lib/db';
import { waitlist } from '@/lib/db/schema';
import { eq } from 'drizzle-orm';
import { addUserAction } from '../users/actions';

export async function approveWaitlistAction(formData: FormData) {
  const id = String(formData.get('id'));
  const [w] = await db.select().from(waitlist).where(eq(waitlist.id, id));
  if (!w) return;
  // Reuse addUserAction so the whitelist/invite branch logic lives in one place.
  const fd = new FormData(); fd.set('email', w.email);
  await addUserAction(fd);
  await db.update(waitlist).set({ approvedAt: new Date() }).where(eq(waitlist.id, id));
}

export async function rejectWaitlistAction(formData: FormData) {
  const id = String(formData.get('id'));
  await db.update(waitlist).set({ rejectedAt: new Date() }).where(eq(waitlist.id, id));
}
```

### Tab 4 — Usage

A single page with: total usage today / this week / this month, a chart of daily totals, and the top 20 users by `usageConsumed`. Useful for spotting runaway use and deciding when to top up everyone.

```tsx
// app/admin/usage/page.tsx
import { db } from '@/lib/db';
import { users } from '@/lib/db/schema';
import { desc, sql } from 'drizzle-orm';
import { UsageOverTimeChart } from '../components';
import { grantToAllAction } from './actions';

export default async function UsageTab() {
  const totals = await db.select({
    granted: sql<number>`sum(${users.usageGranted})`,
    consumed: sql<number>`sum(${users.usageConsumed})`,
  }).from(users);
  const top = await db.select().from(users).orderBy(desc(users.usageConsumed)).limit(20);
  return (
    <section className="space-y-6">
      <div className="stats">
        <div className="stat">
          <div className="stat-title">Granted (all users)</div>
          <div className="stat-value">{totals[0].granted}</div>
        </div>
        <div className="stat">
          <div className="stat-title">Consumed</div>
          <div className="stat-value">{totals[0].consumed}</div>
        </div>
        <div className="stat">
          <div className="stat-title">Remaining</div>
          <div className="stat-value">{Number(totals[0].granted) - Number(totals[0].consumed)}</div>
        </div>
      </div>

      <UsageOverTimeChart />

      <form action={grantToAllAction} className="flex gap-2 items-end">
        <label className="form-control">
          <span className="label-text">Grant +N to every active user</span>
          <input name="amount" type="number" defaultValue={100} className="input input-bordered w-32" />
        </label>
        <button className="btn btn-primary">Apply</button>
      </form>

      <table className="table">
        <thead><tr><th>Email</th><th>Consumed / Granted</th><th>Status</th></tr></thead>
        <tbody>
          {top.map(u => (
            <tr key={u.id}>
              <td>{u.email}</td>
              <td>{u.usageConsumed} / {u.usageGranted}</td>
              <td>{u.deactivatedAt ? 'Deactivated' : 'Active'}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </section>
  );
}
```

For per-day usage you'll need an event log (`usage_events` with `userId`, `createdAt`, `cost`); add it now if you don't have one — it's two columns and a `db.insert` call inside `gatedAiCall`. Aggregate daily totals in `unstable_cache` with a 5-minute revalidate.

### Tab 5 — Notifications

A single page where the founder writes a message and pushes it to all active users or to a hand-picked set. Messages land in the in-app notification surface (the bell icon in the header — see hand-off below). No email is sent from this tab; if email notifications are also wanted, route them through sub-skill 04's email-trigger catalog (e.g., add a "Custom admin announcement" trigger).

**Database schema additions.** Extend `lib/db/schema.ts`:

```ts
export const notifications = pgTable('notifications', {
  id: uuid('id').defaultRandom().primaryKey(),
  subject: text('subject').notNull(),
  body: text('body').notNull(),                       // markdown allowed
  actionUrl: text('action_url'),                      // optional CTA link
  audience: text('audience').notNull(),               // 'all' | 'specific'
  createdAt: timestamp('created_at').defaultNow().notNull(),
  createdBy: text('created_by').notNull().default('admin'),
});

export const notificationRecipients = pgTable('notification_recipients', {
  notificationId: uuid('notification_id').notNull().references(() => notifications.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  readAt: timestamp('read_at'),
}, (t) => ({ pk: primaryKey({ columns: [t.notificationId, t.userId] }) }));
```

**Compose form (`app/admin/notifications/page.tsx`).** A single form on this tab with:

- **Audience selector** — radio buttons "All active users" / "Specific users".
- When "Specific users" is selected: a multi-select component (use `<select multiple>` for MVP, or daisyUI's combobox-style if you have it). The server action queries `users` for active rows. Also include a text input "Filter by email" that filters the multi-select live (use a basic client component for filtering).
- **Subject** — text input, required, max 120 chars.
- **Body** — textarea, required, max 2000 chars, markdown allowed (show a small "Markdown supported" hint).
- **Action URL** — optional text input; renders as a CTA button at the bottom of the rendered notification.
- **Send** button.

Below the form, show **Sent history** — a table of past notifications with subject, audience ("All — 47 users" or "3 users"), sent date, read count (`X / Y read`).

**Server action for sending:**

```ts
'use server';
import { db } from '@/lib/db';
import { users, notifications, notificationRecipients } from '@/lib/db/schema';
import { eq, isNull, inArray } from 'drizzle-orm';

export async function sendNotificationAction(formData: FormData) {
  const subject = String(formData.get('subject')).trim();
  const body = String(formData.get('body')).trim();
  const actionUrl = String(formData.get('actionUrl') ?? '').trim() || null;
  const audience = String(formData.get('audience'));         // 'all' | 'specific'
  const userIds = formData.getAll('userIds').map(String);    // empty if 'all'

  if (!subject || !body) return;

  const [n] = await db.insert(notifications).values({ subject, body, actionUrl, audience }).returning();

  let recipients: { id: string }[] = [];
  if (audience === 'all') {
    recipients = await db.select({ id: users.id }).from(users).where(isNull(users.deactivatedAt));
  } else if (userIds.length) {
    recipients = await db.select({ id: users.id }).from(users).where(inArray(users.id, userIds));
  }

  if (recipients.length) {
    await db.insert(notificationRecipients).values(
      recipients.map(r => ({ notificationId: n.id, userId: r.id }))
    );
  }
}
```

Note: this writes only to the in-app notification surface. If the founder also wants emails to go out, they should pick the matching trigger in sub-skill 04's email-trigger analysis (e.g., add a "Custom admin announcement" trigger to the catalog).

## Hand-off to the header bell icon (sub-skill 02)

The bell icon lives in the global header rule defined by sub-skill 02 (design). This admin sub-skill owns the data layer and the API contract; sub-skill 02 owns the front-end rendering of the bell, the dropdown, and the unread badge.

The bell needs three endpoints, all scoped to the currently signed-in user via `auth()`:

- **Unread count** — `GET /api/notifications/unread-count` returns `{ count: number }`. Cache for 30s on the client.
- **Recent list** — `GET /api/notifications/recent?limit=10` returns the 10 most recent notifications addressed to the current user, each with `readAt` so the client can style unread ones bold.
- **Mark read** — `POST /api/notifications/mark-read` with body `{ notificationId }` (or `{ all: true }` for "mark all read") sets `readAt` on the join row(s).

Stub route handlers:

```ts
// app/api/notifications/unread-count/route.ts
import { auth } from '@/auth';
import { db } from '@/lib/db';
import { notificationRecipients } from '@/lib/db/schema';
import { and, eq, isNull, sql } from 'drizzle-orm';

export async function GET() {
  const session = await auth();
  if (!session?.user?.id) return Response.json({ count: 0 });
  const [row] = await db.select({ count: sql<number>`count(*)::int` })
    .from(notificationRecipients)
    .where(and(eq(notificationRecipients.userId, session.user.id), isNull(notificationRecipients.readAt)));
  return Response.json({ count: row?.count ?? 0 });
}
```

```ts
// app/api/notifications/recent/route.ts
import { auth } from '@/auth';
import { db } from '@/lib/db';
import { notifications, notificationRecipients } from '@/lib/db/schema';
import { and, eq, desc } from 'drizzle-orm';

export async function GET(req: Request) {
  const session = await auth();
  if (!session?.user?.id) return Response.json({ items: [] });
  const limit = Math.min(Number(new URL(req.url).searchParams.get('limit') ?? 10), 50);
  const items = await db.select({
    id: notifications.id,
    subject: notifications.subject,
    body: notifications.body,
    actionUrl: notifications.actionUrl,
    createdAt: notifications.createdAt,
    readAt: notificationRecipients.readAt,
  })
    .from(notificationRecipients)
    .innerJoin(notifications, eq(notifications.id, notificationRecipients.notificationId))
    .where(eq(notificationRecipients.userId, session.user.id))
    .orderBy(desc(notifications.createdAt))
    .limit(limit);
  return Response.json({ items });
}
```

```ts
// app/api/notifications/mark-read/route.ts
import { auth } from '@/auth';
import { db } from '@/lib/db';
import { notificationRecipients } from '@/lib/db/schema';
import { and, eq, isNull } from 'drizzle-orm';

export async function POST(req: Request) {
  const session = await auth();
  if (!session?.user?.id) return new Response('Unauthorized', { status: 401 });
  const { notificationId, all } = await req.json();
  if (all) {
    await db.update(notificationRecipients)
      .set({ readAt: new Date() })
      .where(and(eq(notificationRecipients.userId, session.user.id), isNull(notificationRecipients.readAt)));
  } else if (notificationId) {
    await db.update(notificationRecipients)
      .set({ readAt: new Date() })
      .where(and(
        eq(notificationRecipients.userId, session.user.id),
        eq(notificationRecipients.notificationId, String(notificationId)),
      ));
  }
  return Response.json({ ok: true });
}
```

The header bell consumes these three endpoints; the agent wires the front-end in sub-skill 02 when building the header. If the founder opted out of notifications in the opening DIALOGUE, sub-skill 02 should leave the bell out of the header entirely — `STATE.md # Decisions` is the source of truth.

## Cache aggressively

Aggregation queries are the slow part. `unstable_cache` for 60s on the Overview tab, 5 minutes on Usage. Per-row admin actions invalidate via `revalidatePath('/admin/users')` etc.

## Don't link to it from the public site

Don't add an "Admin" link to the public nav. The founder bookmarks `/admin` themselves. The fewer signals that the route exists, the less attack surface.

## Anti-patterns to avoid

- **Google-Analytics-style dashboards.** The founder wants 5 numbers that decide whether to keep going, not 50 charts that don't.
- **Building before measuring.** If a chart can't be computed because the data isn't captured, instrument first or drop the chart.
- **Hardcoding the admin password.** Always env var. Rotate-able.
- **A weak admin password.** Use `openssl rand -base64 32`. Don't let the user pick something memorable.
- **Skipping the codebase analysis.** A generic dashboard tells the founder nothing they didn't already know.
- **Letting a deactivated user keep a session.** Auth.js's Credentials `authorize` callback already rejects users with `deactivatedAt`; for OAuth/magic-link, also reject in the `signIn` callback.
- **Sending the invite email from a different code path than `lib/email.ts`.** Single helper, single template, single sender.
- **Granting unlimited usage by default.** Set a finite `INITIAL_USAGE_GRANT`. You can always grant more from the dashboard.

## Exit criteria

- `ADMIN_PASSWORD_BOOTSTRAP` was generated, shown once, and used for the first login; the user has since changed it via `/admin/change-password` and the persistent bcrypt hash is stored in `admin_settings.password_hash`. `INITIAL_USAGE_GRANT` is in `.env.local` (and Vercel env after deploy), gitignored.
- Visiting `/admin` without credentials returns 401.
- Visiting `/admin` with the correct password renders the five-tab dashboard (four tabs if notifications were skipped).
- **Overview** renders 4–6 KPIs with real data.
- **Users** lists users; Add User works (sends invite if email is configured, whitelists otherwise); Grant +N works; Deactivate / Reactivate works.
- **Waitlist** lists pending entries; Approve moves the email to `allowedEmails` (and sends an invite if email is configured); Reject works.
- **Usage** shows totals, top-20-by-usage, and a Grant-+N-to-all-active-users form.
- **Notifications** tab works: send to all, send to specific users (multi-select with email filter), sent-history table with read counts.
- DB has `notifications` and `notification_recipients` tables.
- The three header bell endpoints (`/api/notifications/unread-count`, `/api/notifications/recent`, `/api/notifications/mark-read`) exist and return correct data for the signed-in user.
- `STATE.md # Decisions` records whether notifications are enabled (so sub-skill 02 knows whether to render the bell).
- Any new instrumentation is documented under `# Decisions` in `PROJECT.md`.

Move on to `08-analytics.md` (or skip to `09-monetization.md` if analytics aren't in scope for this mode).
