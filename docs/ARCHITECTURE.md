# Architecture

## Overview

```
 Visitors (phones)                          Admins (bureau + city reps)
       │                                            │  Sign in with Google
       ▼                                            ▼
 ┌──────────────────────── Vercel ─────────────────────────────┐
 │  Static SPA (Vite build)      api/index.ts (one function)    │
 │  /, /evenements, /galerie …   /api/content   (public, cached)│
 │                               /api/image/:id/:size (cached)  │
 │                               /api/auth/*    /api/admin/*    │
 └─────────────────────────────────────┬────────────────────────┘
                                       │ OAuth refresh token of the
                                       │ CEBM Google account, scope drive.file
                                       ▼
                    Google account "CEBM" (owned by the association)
                    ├── Google Sheet  "CEBM — Base de données"  (tables)
                    └── Drive folder  "CEBM Site/photos"          (images)
```

Why this shape:
- **Data stays in the association's own Google account.** The bureau can open
  the Sheet, read it, export it, and hand it over to the next bureau.
- **No secrets in the browser.** Only the server talks to Google.
- **Admins sign in with Google**, so there are no passwords to leak or reset.
- **The CDN does the heavy lifting.** Public content and images are cached by
  Vercel, so Google is called rarely, and the site keeps working briefly
  (stale content) even if Google is slow.

Known limits (acceptable at CEBM's size, revisit if it grows a lot):
- Google Sheets is not a real database: no transactions, per-minute API
  quotas, ~1–2 s latency on cold reads. Caching hides this for visitors.
- With the `drive.file` scope the app only sees files **it created**. Photos
  must be uploaded through the admin panel, not dropped into Drive by hand.
  Editing cells of the Sheet by hand is fine (the app created the Sheet).
- Public changes appear within ~60 s (CDN cache). Admin screens are uncached.

## Environments

| Env | Sheet / folder | Who uses it |
|---|---|---|
| Production | prod Sheet + prod photos folder | the public site |
| Staging | separate Sheet + folder in the same account | `vercel dev` locally and Vercel Preview deployments |

Both are created by `scripts/bootstrap-google.ts`. Never seed or test against production.

Google sign-in only works on origins registered in the Google Console
(production domain, `http://localhost`, `http://localhost:3000`). Preview URLs
change per deployment, so admin features are tested locally; previews are for
checking public pages.

## Google Sheet layout

One spreadsheet, one tab per table. Row 1 = headers (exact names below).
Common columns on content tables:
`id` (uuid) · `status` (`draft` | `published` | `archived`) · `version` (int) ·
`createdAt` · `updatedAt` (ISO) · `updatedBy` (admin email).

Lists are stored as comma-separated ids (`photoIds`). Booleans as `TRUE`/`FALSE`.

| Tab | Columns (besides the common ones) |
|---|---|
| `events` | title, city, date (YYYY-MM-DD), time (HH:mm), location, description, photoIds |
| `announcements` | title, message, date, category (`general`/`bourse`/`administratif`/`opportunite`), pinned, photoIds |
| `gallery` | title, description, date, city, category (`evenements`/`culture`/`bourses`/`archives`), photoIds |
| `responsables` | name, role, city, isNational, email, phone, showPhone, sortOrder |
| `cities` | slug, name, color, sortOrder, active |
| `settings` | key, value (contact email, WhatsApp link, phone, notes…) — no common columns |
| `photos` | id, fullFileId, thumbFileId, alt, width, height, createdAt, uploadedBy |
| `admins` | email, name, role (`super`/`national`/`city`), city, active, createdAt — tab protected (owner-only edit) |
| `audit_log` | at, adminEmail, action, table, rowId, summary — append-only |
| `guides` *(phase 2)* | slug, title, city (or `all`), category, body (Markdown), sortOrder |
| `opportunities` *(phase 2)* | title, type (`stage`/`bourse`/`emploi`/`formation`), organization, city, deadline, link, description |
| `messages` *(phase 3)* | name, contact, city, subject, message, handled — never public |

The zod schema for each table lives in `shared/schemas.ts` and is the single
source of truth. When you add a column: update the schema, the bootstrap script
(headers), and this table.

## Drive layout

```
CEBM Site/
├── production/photos/
└── staging/photos/
```
Each uploaded photo = 2 files (`full` ≤ 1600 px, `thumb` ≤ 480 px), created by
the app, private (not shared publicly). The browser never sees Drive IDs; it
uses `/api/image/<photoId>/<size>`.

## API

Single function `api/index.ts` with the Web `fetch` handler; `vercel.json`
rewrites `/api/(.*)` → `/api`, and the router reads the original pathname from
`request.url`. (If a Vercel change ever stops preserving the pathname, rewrite
to `/api?route=$1` instead.)

### Public
| Method | Path | Notes |
|---|---|---|
| GET | `/api/health` | `{ ok: true }`, no Google call |
| GET | `/api/content` | all published content + cities + public settings. `s-maxage=60, stale-while-revalidate=600` |
| GET | `/api/image/:photoId/:size` | `size` ∈ `thumb`,`full`; photoId must exist in `photos`; `immutable` 1-year cache |

### Auth
| Method | Path | Notes |
|---|---|---|
| POST | `/api/auth/google` | body `{ credential }` (GIS ID token). Verify signature + `aud` = client id + `email_verified`; email must be an active row in `admins` (or in `BOOTSTRAP_ADMIN_EMAILS`). Sets session cookie. |
| GET | `/api/auth/me` | `{ email, name, role, city }` or 401 |
| POST | `/api/auth/logout` | clears cookie |

Session: JWT (HS256, `SESSION_SECRET`) with `{ sub: email, exp: 8h }` in
`__Host-cebm_session` (`HttpOnly; Secure; SameSite=Strict; Path=/`).
On every admin request the role and `active` flag are re-read from the
`admins` tab (cached ≤ 60 s), so removing an admin takes effect within a minute.

### Admin (all `Cache-Control: no-store`)
| Method | Path | Roles |
|---|---|---|
| GET | `/api/admin/:table` | any admin (city admins get all rows but can edit only their city) |
| POST | `/api/admin/:table` | per table, see below |
| PATCH | `/api/admin/:table/:id` | body includes `version` → 409 on mismatch |
| POST | `/api/admin/:table/:id/archive` | soft delete |
| POST | `/api/admin/photos` | multipart: `full`, `thumb`, `alt` → creates `photos` row |
| GET/POST/PATCH | `/api/admin/admins…` | `super` only |
| GET | `/api/admin/audit` | `super` only |

Role matrix:

| | super | national | city |
|---|---|---|---|
| events, gallery | all | all | own city only |
| announcements, guides, opportunities | all | all | read only |
| responsables | all | all | own city, non-national only |
| cities, settings | ✔ | ✔ | ✘ |
| admins, audit log | ✔ | ✘ | ✘ |

## Front-end

- `src/lib/api.ts`: typed fetch wrapper; parses responses with `shared/` schemas.
- `ContentProvider` loads `/api/content` once, keeps it in memory, refreshes
  in the background when the tab regains focus.
- `AuthProvider` (admin only) calls `/api/auth/me`; redirects to `/admin/login` on 401.
- Image component `<Photo id size alt />` builds `/api/image/...` URLs.
- Theme: `public/theme-init.js` sets `data-theme` before paint; toggle stores
  the preference in `localStorage` (try/catch).

## Environment variables

See `.env.example`. Server-only unless prefixed `VITE_`.

| Var | Where | Purpose |
|---|---|---|
| `GOOGLE_CLIENT_ID` / `VITE_GOOGLE_CLIENT_ID` | server / client | OAuth Web client id (public value) |
| `GOOGLE_CLIENT_SECRET` | server | OAuth client secret |
| `GOOGLE_REFRESH_TOKEN` | server | CEBM account token, scope `drive.file` |
| `SHEET_ID` | server | spreadsheet of this environment |
| `DRIVE_PHOTOS_FOLDER_ID` | server | photos folder of this environment |
| `SESSION_SECRET` | server | ≥ 32 random bytes, different per environment |
| `BOOTSTRAP_ADMIN_EMAILS` | server | comma list of super admins before the `admins` tab is filled |
| `SITE_ORIGIN` | server | e.g. `https://cebm-maroc.org` (Origin check) |
| `TURNSTILE_SECRET` / `VITE_TURNSTILE_SITE_KEY` | server / client | phase 3 public forms |
