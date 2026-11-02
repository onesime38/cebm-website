# Security

## What we protect, from whom

| Asset | Threat | Main control |
|---|---|---|
| Admin panel | Anyone reading the JS bundle / GitHub repo (the old password `cebm2024` is public forever in git history) | No passwords at all: Google sign-in + allow-list in the `admins` tab, checked on the server |
| CEBM Google account | Stolen refresh token | Token only in Vercel env vars; scope `drive.file` (only files the app created); 2-step verification on the account |
| Members' personal data | Scraping, curiosity, harassment | Publish the minimum; phones only with `showPhone` consent; no public scholarship lookup; messages/members never on public endpoints |
| Content integrity | A city admin editing other cities, a former admin | Role + city checks on the server; roles re-read from the Sheet every minute; audit log |
| Visitors | XSS through admin-entered content or links | React escaping, no `dangerouslySetInnerHTML`, URL allow-list, strict CSP |
| Sheet | Formula injection (`=IMPORTXML(…)` sending data out) | All writes `RAW` |
| Availability / quota | Spam on public endpoints | CDN caching; Turnstile + rate limits on public forms (phase 3) |

## Controls checklist (must all be true before launch)

**Authentication & sessions**
- [ ] `ADMIN_CREDENTIALS`, `cebm_admin` localStorage flag and the `#admin` hash shortcut are removed.
- [ ] Login only via Google Identity Services; server verifies ID token signature, `aud`, `iss`, `exp`, `email_verified`.
- [ ] Email must be `active` in `admins` (or in `BOOTSTRAP_ADMIN_EMAILS`).
- [ ] Session cookie `__Host-cebm_session`: `HttpOnly; Secure; SameSite=Strict; Path=/`, 8 h expiry, signed with `SESSION_SECRET`.
- [ ] Logout clears the cookie; removing an admin in the Sheet blocks them within 60 s.

**Authorization**
- [ ] Every `/api/admin/*` handler starts with `requireRole`.
- [ ] City scoping enforced server-side (tests cover a city admin trying another city → 403).
- [ ] `admins` and `audit_log` endpoints are `super`-only.

**Input & output**
- [ ] zod `.strict()` validation on every body/query/param; max lengths on every string (title ≤ 150, text ≤ 5,000…).
- [ ] JSON bodies ≤ 100 KB, uploads ≤ 3 MB, image type checked by magic bytes (JPEG/WebP).
- [ ] URLs: `https:` only; WhatsApp links only `https://chat.whatsapp.com/…` or `https://wa.me/…`; `mailto:`/`tel:` built by code from validated values.
- [ ] Sheets writes use `RAW`.
- [ ] `/api/content` whitelists fields per table (no `updatedBy`, no hidden phones, no messages/admins/members).
- [ ] Image route serves only ids found in `photos`.
- [ ] Errors: generic French message to the client, details only in server logs; no tokens or bodies logged.

**Browser hardening (`vercel.json` headers on all routes)**
```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://accounts.google.com/gsi/client;
  style-src 'self' 'unsafe-inline' https://accounts.google.com/gsi/style;
  img-src 'self' data: blob: https://*.googleusercontent.com;
  font-src 'self';
  connect-src 'self' https://accounts.google.com/gsi/;
  frame-src https://accounts.google.com/gsi/;
  frame-ancestors 'none'; base-uri 'self'; form-action 'self'; object-src 'none';
  upgrade-insecure-requests
Cross-Origin-Opener-Policy: same-origin-allow-popups
Strict-Transport-Security: max-age=63072000; includeSubDomains
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=()
```
(The CSP is written on one line in `vercel.json`. When Turnstile is added in
phase 3, add `https://challenges.cloudflare.com` to `script-src` and `frame-src`.)
- [ ] No inline `<script>` in `index.html` (theme bootstrap moved to `public/theme-init.js`).
- [ ] Check headers on a Preview deployment with https://securityheaders.com and the browser console (no CSP errors, Google button works).

**Secrets & supply chain**
- [ ] Only `VITE_GOOGLE_CLIENT_ID` and `VITE_TURNSTILE_SITE_KEY` are exposed to the client.
- [ ] `.env*` ignored (except `.env.example`); GitHub secret scanning on; Dependabot alerts + weekly updates on.
- [ ] `npm audit` has no high/critical issues.
- [ ] Different `SESSION_SECRET`, Sheet and folder for staging and production.

**Personal data** (Morocco: loi 09-08, supervised by the CNDP)
- [ ] Public scholarship status lookup removed (keep only the CESAM "code bourse" link).
- [ ] Written OK from every person whose name/phone is published; `showPhone` defaults to `FALSE`.
- [ ] `/confidentialite` page: what data is collected, why, who can see it, how to ask for removal (contact email).
- [ ] Fake/sample people never appear in production.

## Review checklist (used by `/audit` on every PR)

1. New env var? Is it server-only? Added to `.env.example` and docs?
2. New/changed endpoint: method check, Origin check (mutations), `requireRole`, city scope, zod `.strict()`, size cap, audit log, generic errors?
3. Anything new returned by `/api/content`? Is it meant to be public?
4. Any user-controlled value rendered as HTML, used in `href`/`src`, or written to the Sheet without `RAW`?
5. Any new third-party script, font, or domain? CSP updated deliberately?
6. Tests added for the unhappy paths?
7. Dependencies added: necessary, maintained, `npm audit` clean?

## If something goes wrong

| Incident | Do this |
|---|---|
| Refresh token leaked | In the CEBM Google account: Security → third-party access → remove "CEBM Site". Re-run `scripts/get-refresh-token.ts`, update Vercel env, redeploy. |
| Admin's phone/laptop lost, or admin leaves | Set `active = FALSE` in `admins` (effective ≤ 60 s). |
| Suspected forged sessions | Rotate `SESSION_SECRET` in Vercel and redeploy (logs everyone out). |
| Bad content published / vandalism | Check `audit_log`; restore the row (archived rows are kept); Sheet version history (File → Version history) as last resort. |
| CEBM Google account compromised | Recover the account, change password, review 2-step verification, then do the refresh-token steps above. |

Keep at least two bureau members able to recover the CEBM Google account.
