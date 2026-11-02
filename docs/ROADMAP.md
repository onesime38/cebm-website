# Roadmap

One task = one branch = one PR. Tick `- [x]` when every acceptance criterion
passes. `(H)` = done by a human, not Claude. Tasks marked `⇄` can run in
parallel with the previous one (good for splitting work between the two of you).

---

## Phase 0 — Clean foundations (no visible behavior change)

- [x] **0.1 Repo cleanup + tooling**
  - Delete `build-errors*.txt`, `src/App.css`, unused `src/assets/{react,vite}.svg`, `hero.png`, `logocebem.png` (after confirming they are unused), template `README.md`.
  - Add scripts `typecheck` and `test` (vitest), `.env.example`, `.nvmrc` (Node 22).
  - New `README.md` (French): what the site is, how to run it, link to `docs/`.
  - ✅ `npm run lint && npm test && npm run build` pass; no unused files.

- [ ] **0.2 Split `App.tsx`** (one person, merge fast — everything else depends on it)
  - Move types to `shared/` (zod schemas), pages to `src/pages/`, admin to `src/admin/`, shared UI to `src/components/`, helpers to `src/lib/`.
  - Replace the 300 KB `LOGO_B64` string with an imported image file.
  - Move repeated inline styles to CSS classes + tokens (at least layout, cards, buttons, forms).
  - ✅ Site looks and behaves the same (compare screenshots at 360px and 1366px, light + dark). No file > ~300 lines. JS bundle smaller.

- [ ] **0.3 Real URLs** ⇄
  - React Router with the routes listed in `.claude/rules/frontend.md`; 404 page; `document.title` per page.
  - `vercel.json` with the SPA rewrite.
  - ✅ Refresh on `/evenements/<id>` shows that event; back button works; unknown URL → 404 page.

- [ ] **0.4 HTML head, fonts, accessibility basics** ⇄
  - `index.html`: `lang="fr"`, real title, meta description, Open Graph + Twitter tags with the logo, theme-color; theme script moved to `public/theme-init.js`.
  - Self-host Playfair Display (and the body font) with `@fontsource`.
  - Clickable cards become links; focus styles; lightbox closes on Escape.
  - ✅ Sharing the home URL on WhatsApp shows title + logo; Lighthouse accessibility ≥ 90.

- [ ] **0.5 Known bugs** ⇄
  - Upcoming events: compare dates as strings in Africa/Casablanca, sort ascending, then take 6.
  - `announcements.sort` mutating props → copy first.
  - Stats: count real cities in use; remove hardcoded "5+ années" or make it a setting.
  - Admin on mobile: tab bar scrolls/wraps instead of being cut; remove the empty dark header block.
  - ✅ Unit tests for the date helper (today's event is shown; past events hidden).

---

## Phase 1 — Backend + security (required before launch)

- [ ] **1.1 (H) Google + Vercel setup** — follow `docs/GOOGLE_SETUP.md` for staging *and* production.
  - ✅ You have: client id/secret, refresh token, `SESSION_SECRET`s, and the app is "In production" in Google.

- [ ] **1.2 Server foundation**
  - `server/env.ts`, `server/google.ts`, `server/sheets.ts` (`Table<T>`), `server/http.ts`, `shared/schemas.ts` for every table in ARCHITECTURE.
  - `api/index.ts` router + `GET /api/health`.
  - ✅ vitest covers row↔object mapping, RAW writes, schema rejection of bad rows, missing env var error.

- [ ] **1.3 Google scripts**
  - `scripts/get-refresh-token.ts` (local loopback OAuth flow, scope `drive.file`, prints token once).
  - `scripts/bootstrap-google.ts --env staging|production` creates the Sheet (tabs + headers + protected `admins`/`audit_log` ranges) and Drive folders, prints the IDs. Idempotent: refuses to overwrite.
  - `scripts/seed.ts` fills **staging only** with realistic sample data (refuses if `SHEET_ID` looks like production).
  - ✅ (H) runs them; IDs end up in `.env.local` and Vercel.

- [ ] **1.4 Public content from the API**
  - `GET /api/content` with field whitelists + CDN cache headers.
  - Front-end `ContentProvider`; delete `useLocalStorage` for content and all `INITIAL_*` constants.
  - Loading and error states.
  - ✅ A row edited by hand in the staging Sheet appears on the site within ~1 min, on another device.

- [ ] **1.5 Google sign-in + sessions**
  - GIS button on `/admin/login`; `/api/auth/google|me|logout`; `requireRole`; bootstrap admins.
  - Remove `ADMIN_CREDENTIALS`, the `cebm_admin` flag and the `#admin` hash route.
  - ✅ Tests: unknown email → 403; inactive admin → 403; tampered cookie → 401; no cookie → 401. Grep finds no `cebm2024`.

- [ ] **1.6 Admin CRUD through the API**
  - Events, announcements, gallery, responsables, cities, settings: list/create/edit/archive.
  - Optimistic concurrency (`version`), audit log, city scoping.
  - ✅ Two browsers editing the same event → second save gets the 409 message. City admin cannot edit another city (UI hides it *and* API returns 403).

- [ ] **1.7 Photos on Drive**
  - Client resize to `full`/`thumb`; `POST /api/admin/photos`; `GET /api/image/:id/:size`; `<Photo>` component.
  - ✅ Uploading a 10 MB phone photo works; the stored files have no EXIF GPS; grid pages load thumbs only; unknown id → 404.

- [ ] **1.8 Hardening**
  - Security headers + CSP in `vercel.json` (exact values in `docs/SECURITY.md`); Origin check; body caps; generic errors.
  - ESLint `no-restricted-imports` (src ✘ server); Dependabot config; `npm audit` clean.
  - ✅ securityheaders.com grade A on a preview; no CSP errors in console; Google sign-in still works.

- [ ] **1.9 Privacy**
  - `/bourse` keeps only the CESAM code-bourse link + useful explanations; remove matricule lookup and scholarship data.
  - `showPhone` on responsables (default false); `/confidentialite` page; footer link.
  - ✅ `/api/content` contains no phone where `showPhone` is false, no matricules, no admin emails.

- [ ] **1.10 Admin management**
  - Super admin UI: add/deactivate admins, set role + city; manage cities; read audit log.
  - ✅ Deactivated admin is logged out within 60 s.

- [ ] **1.11 Launch**
  - Run `/audit` and Claude Code's `/security-review`; fix findings.
  - (H) Production env vars in Vercel, real content entered, consent from listed people, domain connected.
  - ✅ Full flow tested on production with a real phone: sign in, add event with photo, see it publicly, archive it.

---

## Phase 2 — High-value features

- [ ] **2.1 Guide du nouvel arrivant** — `guides` table; pages `/guide` and `/guide/:slug`; Markdown rendered with `react-markdown` (no raw HTML); filter by city; admin editor with preview.
  Starter topics (content written by the bureau): carte de séjour, AMCI/CESAM bourse & code bourse, ouvrir un compte bancaire, carte SIM & internet, logement, santé/assurance, transport, contacts utiles par ville.
- [ ] **2.2 Share events** — "Partager sur WhatsApp" (`https://wa.me/?text=`), "Ajouter au calendrier" (.ics download) and Google Calendar link; copy-link button.
- [ ] **2.3 Opportunités** — `opportunities` table; list with type/city filters; deadline badge; expired items hidden automatically; admin CRUD.
- [ ] **2.4 Rich link previews per event** (optional) — rewrite `/evenements/:id` to a route that returns `index.html` with that event's OG title/description/photo injected server-side (WhatsApp does not run JS).
- [ ] **2.5 Installable app (PWA)** — manifest, icons, service worker caching `/api/content` and thumbnails for offline reading.

## Phase 3 — Community

- [ ] **3.1 Contact / suggestion form** — `messages` table; Cloudflare Turnstile + honeypot + per-IP rate limit; optional anonymous mode; admin inbox with "traité" flag.
- [ ] **3.2 Documents** — statuts, PV de réunions, rapports financiers: PDF upload to Drive, public or admin-only per document.
- [ ] **3.3 Member registration** — admin-approved; consent checkbox; data visible only to admins; fields limited to what the bureau really needs (name, city, university, field, year, WhatsApp). Check the 09-08 obligations before launching this one.
- [ ] **3.4 Notifications** — link the official WhatsApp channel; optional email digest of new announcements.
- [ ] **3.5 Analytics** — Vercel Web Analytics (no cookies), pages and events only.
