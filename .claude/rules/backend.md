---
paths:
  - "api/**/*.ts"
  - "server/**/*.ts"
  - "shared/**/*.ts"
  - "scripts/**/*.ts"
---

# Backend rules (api/, server/, shared/, scripts/)

Read `docs/ARCHITECTURE.md` before changing routes, the Sheet layout, or auth.

## Structure
- `api/index.ts` stays tiny: parse URL → call a handler in `server/routes/*` →
  catch errors → `Response`. Do not add more files under `api/` (Vercel Hobby
  allows 12 functions per deployment and Vercel recommends one shared function).
- `vercel.json` rewrites `/api/(.*)` to `/api`. Route on `new URL(request.url).pathname`.
- `shared/` contains only zod schemas, types and pure helpers — no `process`,
  no `fetch`, no DOM. The client may import it.
- `server/` is never imported by `src/`. ESLint should enforce this
  (`no-restricted-imports`).

## Every handler
1. Method check → 405 otherwise.
2. For POST/PUT/PATCH/DELETE: `assertSameOrigin(request)` (Origin header must
   equal `SITE_ORIGIN`, or `http://localhost:3000` when `NODE_ENV !== "production"`).
3. Admin routes: `const admin = await requireRole(request, ["super", "national", "city"])`.
   City admins may only create/edit/delete rows where `row.city === admin.city`;
   enforce it in the handler, not only in the UI.
4. Parse body with a size cap (JSON ≤ 100 KB; uploads ≤ 3 MB) and validate with
   the zod schema from `shared/` using `.strict()`.
5. Return `json(data, status)` from `server/http.ts`. Errors go through
   `HttpError(status, publicMessage)`; unknown errors → 500 "Erreur interne".
6. Write an audit-log row for every create/update/archive/delete
   (`audit_log`: at, adminEmail, action, table, rowId, summary).

## Google Sheets
- Access only through `server/sheets.ts` (`Table<T>` with list/get/insert/update/archive).
- Row 1 is the header row; map columns by header name, never by index.
- Writes: `valueInputOption: "RAW"`. Read with `valueRenderOption: "UNFORMATTED_VALUE"`.
- Validate every row read from the Sheet with zod; skip invalid rows and
  `console.warn` the row id (humans can edit the Sheet by hand and make mistakes).
- Optimistic concurrency: rows have `version`. Updates must send the version
  they edited; mismatch → 409 "Ce contenu a été modifié par quelqu'un d'autre".
- Deletion = set `status: "archived"` (recoverable). Hard delete only for
  `photos` that are no longer referenced, and only from the super admin.
- Cache reads in memory per instance for ≤ 30 s; invalidate on write.

## Google Drive
- OAuth scope is `drive.file` only. The app can only see files **it created**.
  Never ask for `drive` or `spreadsheets` scopes (restricted/sensitive → Google
  verification). If something seems to need them, stop and ask.
- Uploads: accept `image/jpeg` and `image/webp` only, check magic bytes, ≤ 3 MB.
  The client already resizes; the server still verifies.
- Image route: `GET /api/image/:photoId/:size` (`size` = `thumb` | `full`).
  Look up the photo in the `photos` table → stream the Drive file →
  `Cache-Control: public, max-age=31536000, immutable`. Unknown id → 404.

## Public content route
- `GET /api/content` returns only `status === "published"` rows, with fields
  whitelisted per table (never `updatedBy`, emails of admins, `showPhone=false`
  phone numbers, or internal notes).
- `Cache-Control: public, s-maxage=60, stale-while-revalidate=600`.

## Secrets
- Read env only through `server/env.ts` (zod-validated, throws at startup with
  the *name* of the missing var, never its value).
- Never log tokens, cookies, ID tokens, refresh tokens, or full request bodies.

## Tests (vitest)
- Unit-test: zod schemas, row ↔ object mapping, URL validators, `requireRole`
  (city scoping), origin check, session sign/verify, upload magic-byte check.
- Mock Google clients; tests must never hit real Google APIs.
