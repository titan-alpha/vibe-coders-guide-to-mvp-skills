# 17 · Ship Checklist

Goal: final go/no-go before the user shares the URL with another human.

## AUTONOMOUS — run the checklist

Tick each one. Anything that fails goes back to the relevant sub-skill.

### Functional

- [ ] The MVP slice works end-to-end on the production URL.
- [ ] Auth (if used) works on production, including sign-out.
- [ ] At least one happy-path and one error-path are tested by hand.

### Trust signals

- [ ] Page `<title>` and `<meta name="description">` are real, not the framework defaults.
- [ ] Favicon is set and isn't the framework default.
- [ ] Open Graph image and tags render correctly (test with https://www.opengraph.xyz/).
- [ ] No Lorem Ipsum, TODO, or "Welcome to Next.js" anywhere.

### Versioning (verifies SKILL.md semver rule + sub-skill 14)

- [ ] `package.json` `version` is **not** the framework default (typically `0.1.0` from `create-next-app`). It's been bumped to reflect actual releases.
- [ ] The version was bumped from the previous deploy. Check:
      ```bash
      LAST=$(git describe --tags --abbrev=0 --match='v*' 2>/dev/null)
      CURR=$(node -p "require('./package.json').version")
      echo "previous tag: $LAST"
      echo "current package.json: v$CURR"
      ```
      If `v$CURR` matches `$LAST`, the bump didn't happen — go back to sub-skill 14.
- [ ] A matching `vX.Y.Z` git tag exists for the current `package.json` version:
      ```bash
      git tag -l "v$(node -p "require('./package.json').version")"
      ```
      Expected: a single line with the tag name. If empty, run `git tag -a v$VERSION -m v$VERSION && git push --follow-tags`.
- [ ] `CHANGELOG.md` has a section for the current version with at least one user-visible bullet (Added / Changed / Deprecated / Removed / Fixed / Security).
- [ ] `STATE.yaml`'s `decisions.released_versions` has an entry for this version with date + URL.

### Hygiene

- [ ] `.env.local` is gitignored and never committed.
- [ ] Production has its own secrets in Vercel env vars, not the dev ones.
- [ ] `npm audit --omit=dev` is clean (or known issues documented).
- [ ] Lighthouse mobile scores still ≥ 90 / 95 / 95 / 90.

### Logging hygiene (verifies the SKILL.md logging rule + sub-skill 11)

- [ ] `LOG_LEVEL` in production env vars is `info` or `warn` &mdash; never `debug`. Verify via Vercel project env settings or platform equivalent.
- [ ] No `console.log` calls remain in `app/`, `lib/`, or `pages/`. Run:
      ```bash
      grep -rEn '\bconsole\.(log|debug)\b' app lib pages 2>/dev/null
      ```
      Expected: empty. If anything matches, route it through `lib/log.ts` instead.
- [ ] Recent production logs contain no leaked secrets. Pull the last ~500 lines from Vercel / your log host and grep for the patterns the redactor was meant to strip:
      ```bash
      vercel logs --since=1h | grep -E 'sk_(test|live)_|re_|whsec_|Bearer\s+[A-Za-z0-9]|\$2[aby]\$\d{2}\$|password.*[:=]\s*[^\s,}]+'
      ```
      Expected: empty. **If anything matches: rotate the leaked credential immediately, fix the offending log line, and re-deploy before sharing the URL.** This is a release-blocker.
- [ ] Recent production logs contain no raw email addresses, phone numbers, or other PII. Spot-check by grepping for `@` and a few sample digits-of-phone patterns. Bulk leakage means a redact rule is missing.

### Security verification (verifies sub-skill 11)

- [ ] Security headers grade is **A or A+** at https://securityheaders.com/?q=&lt;your-domain&gt; (re-test after deploy &mdash; the headers must be present in the production response, not just in dev).
- [ ] HSTS header is set with `preload` directive. Verify:
      ```bash
      curl -sI https://&lt;your-domain&gt; | grep -i strict-transport-security
      # Expect: Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
      ```
      (Optional but recommended: submit at https://hstspreload.org/ once the site has been on HTTPS for 30+ days.)
- [ ] CSP is non-trivial (not `default-src *` or absent). Verify with the same `curl -sI` &mdash; the `Content-Security-Policy` header should name specific origins per directive.
- [ ] TLS 1.3 is in use. Verify:
      ```bash
      curl -v https://&lt;your-domain&gt; 2>&1 | grep -i 'SSL connection'
      # Expect: SSL connection using TLSv1.3 / ...
      ```
- [ ] Rate limits are active on every public endpoint that triggers cost (auth, AI, email send, DB writes). Pick one endpoint and curl it in a loop:
      ```bash
      for i in {1..30}; do curl -s -o /dev/null -w '%{http_code}\n' -X POST https://&lt;your-domain&gt;/api/&lt;limited-route&gt; -H 'Content-Type: application/json' -d '{}'; done | sort | uniq -c
      # Expect: a mix of 200/4xx and at least one 429 well before 30 requests.
      ```
      Confirm the `429` response includes `Retry-After`.
- [ ] Disk-level encryption is on for the production database. Confirm in the provider console:
      - Neon → Project &rarr; Settings &rarr; Encryption: AES-256, KMS-managed.
      - Supabase → Settings &rarr; Database &rarr; "Encryption at rest: enabled".
      - Vercel Postgres → enabled by default (no toggle).
      - Self-managed: `select * from pg_settings where name='data_encryption';` or AWS RDS console &rarr; Storage &rarr; Encryption.
- [ ] Application-level encryption is wired for any column the agent classified as PII / regulated (sub-skill 11's `lib/crypto.ts`). If the project has none of those columns, write that explicitly: *"No app-level encryption needed &mdash; no columns classified as PII / regulated by sub-skill 03."*
- [ ] No production secrets are referenced in client-side code. Quick check:
      ```bash
      grep -rEn '(process\.env\.STRIPE_SECRET|process\.env\.OPENAI_API_KEY|process\.env\.RESEND_API_KEY|process\.env\.AUTH_SECRET|process\.env\.APP_ENCRYPTION_KEY)' app components --include='*.tsx' --include='*.ts' | grep -v 'use server\|server-only'
      ```
      Anything that matches needs to move into a server action / route handler.

### Resilience

- [ ] A 404 page exists and is on-brand.
- [ ] At least one server-side error path returns a useful message instead of a stack trace.
- [ ] Forms show inline errors, not browser default popups.

### Compliance verification (verifies sub-skill 03)

- [ ] `STATE.yaml # Decisions` has the **vertical classification** (B2C general / B2C regulated / B2B small / B2B Enterprise) and the list of frameworks the agent surfaced.
- [ ] If the project is in a regulated vertical (HIPAA / FERPA / GLBA / PCI-DSS), the user's explicit acknowledgment and chosen path (implement-now / pivot-scope / defer-with-risk) is recorded.
- [ ] If geographic compliance choice was **(b) IP-block**, verify the middleware is live: `curl -H 'x-vercel-ip-country: GB' https://<your-domain>/` returns `451 Unavailable for Legal Reasons` with the polite-explanation page.
- [ ] If geographic compliance choice was **(a) comply**, the privacy policy lists every sub-processor with a signed DPA (OpenAI, Resend, Vercel, Stripe, Upstash, etc.) and the data-export + delete endpoints work end-to-end (test with a real test account).

### Legal/social (only if relevant)

- [ ] If you collect emails: a one-line privacy note exists ("we use your email to log you in; we don't share it").
- [ ] If you take payment in v1: you're using Stripe Checkout, not rolling your own form.

## DIALOGUE — wrap up with the user

Tell the user, in this order:

1. **The URL.** Production link, repo link, and how to redeploy (`git push` triggers a Vercel deploy).
2. **What's done.** One paragraph summarizing what you built together.
3. **The next 3 things.** Suggest 3 concrete next steps, in the order you'd do them. Examples: "Buy a custom domain and connect it in Vercel" · "Add Vercel Analytics to see who's visiting" · "Ask 5 friends from the target audience to try it and write down what confused them."

## Exit criteria

- All checklist items are ticked or explicitly marked as not applicable.
- The user has the URL, the repo, and the next-steps list in their hands.

Move on to `18-deliverables.md`.
