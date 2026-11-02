# CEBM — Communauté des Étudiants Burundais au Maroc

Public website + admin panel for the Burundian student community in Morocco.
Visitors read events, announcements, gallery, contacts and guides. Admins (the
CEBM bureau and city representatives) publish that content.

Two developers work on this repo with Claude Code. Treat every change as code
someone else will review and maintain.

## Current state → target state

- **Today:** single-page React app, everything in `src/App.tsx` (~2,300 lines),
  data in `localStorage` (so admin edits never reach visitors), hardcoded admin
  password in the bundle. **Do not build on that pattern.**
- **Target:** React SPA + one Vercel Function API. Data lives in the CEBM Google
  account: a Google Sheet is the database, a Drive folder stores photos.
  Admins sign in with Google. See `docs/ARCHITECTURE.md`.
- Work is split into phases in `docs/ROADMAP.md`. Do the next unchecked task,
  not a random one. Tick the checkbox when its acceptance criteria pass.

## Stack

- Vite + React 19 + TypeScript (strict), React Router (`react-router`) for URLs
- Vercel: static SPA + **one** function `api/index.ts` (Web `fetch` handler)
  that routes by pathname. Shared backend code lives in `server/`, never in `api/`.
- Google: `@googleapis/sheets`, `@googleapis/drive`, `google-auth-library`
  (not the full `googleapis` package — too heavy for cold starts)
- Validation: `zod` schemas in `shared/` — used by both server and client
- Sessions: `jose` (signed JWT in an httpOnly cookie)
- Tests: `vitest`
- Fonts self-hosted with `@fontsource/*` (no Google Fonts CDN)

## Commands

```bash
npm install
vercel dev            # full stack locally on http://localhost:3000 (SPA + /api)
npm run dev           # front-end only (no API)
npm run build         # tsc -b && vite build — must pass before any commit
npm run lint          # eslint — must pass before any commit
npm test              # vitest run — must pass before any commit
npm run typecheck     # tsc -b --noEmit (add this script in phase 0)
```

`vercel dev` needs `vercel login` + `vercel link` once, and a `.env.local`
(copy `.env.example`). Never run scripts that write to the **production** Sheet
unless the human explicitly says so; use the staging Sheet.

## Folder layout (target)

```
api/index.ts          the only Vercel Function: router → server/routes/*
server/               backend only (Node). Never imported from src/
  env.ts              zod-validated process.env (fail fast on missing vars)
  google.ts           OAuth2 client from refresh token; sheets + drive clients
  sheets.ts           generic table repo (list/get/insert/update/archive)
  drive.ts            photo upload / stream / trash
  auth.ts             Google ID-token verify, session cookie, requireRole()
  http.ts             json(), errors, origin check, body-size limit
  routes/             content.ts, image.ts, auth.ts, admin.ts …
shared/               zod schemas + inferred types (no Node or DOM APIs)
src/
  main.tsx, router.tsx
  pages/              public pages (one file per route)
  admin/              admin pages + forms
  components/         reusable UI
  lib/                api client, image compression, dates, theme
  styles/             CSS (tokens in variables.css)
scripts/              one-off Node scripts (refresh token, bootstrap, seed)
docs/                 ARCHITECTURE, ROADMAP, SECURITY, GOOGLE_SETUP, WORKFLOW (humans)
```

## Non-negotiable security rules

Full checklist: `docs/SECURITY.md`. These apply to every change:

1. **No secrets in the client.** Anything prefixed `VITE_` ships to browsers.
   Only `VITE_GOOGLE_CLIENT_ID` and `VITE_TURNSTILE_SITE_KEY` may be `VITE_`.
   Never hardcode credentials, tokens, sheet IDs of private data, or passwords.
2. **Never read, print, or commit `.env*` files** (except `.env.example`).
3. **Every authorization check is on the server.** Hiding a button is not
   security. Each admin route calls `requireRole(...)` first.
4. **Validate every input on the server with zod** (body, query, params), even
   if the client already validated. Reject unknown fields (`.strict()`).
5. **Sheets writes use `valueInputOption: "RAW"`** — never `USER_ENTERED`
   (prevents formula injection like `=IMPORTXML(...)`).
6. **Image proxy only serves photo IDs listed in the `photos` sheet.** Never
   proxy an arbitrary Drive file ID from the query string.
7. **No `dangerouslySetInnerHTML`.** User-supplied URLs must pass the
   allow-list validator in `shared/` (https only; WhatsApp links only from
   `chat.whatsapp.com` / `wa.me`). Never put user input in `href` unvalidated.
8. **Mutating requests** (POST/PUT/PATCH/DELETE) must pass the Origin check.
   The session cookie is `__Host-cebm_session`, `HttpOnly; Secure; SameSite=Strict; Path=/`.
9. **Personal data is minimal and opt-in.** Do not add features that publish
   phone numbers, matricules, scholarship amounts or student lists publicly.
   Phone numbers on the site require `showPhone = true` on that record.
10. **Errors:** log details server-side, return generic messages to clients.
    Never return stack traces or Google API error bodies.

If a task seems to require breaking one of these, stop and ask the human.

## Conventions

- **UI language is French** (with correct accents). Code, identifiers and
  comments are in English.
- Commits: Conventional Commits, message in French, as the repo already does:
  `feat(events): ajouter le partage WhatsApp`, `fix(auth): …`, `chore: …`.
- One concern per commit. Never commit generated files, `dist/`, logs, or
  `build-errors*.txt`.
- Branch per roadmap task: `phase-1/backend-sheets`, etc. Never push to `main`
  directly and never force-push.
- TypeScript strict; no `any` (use `unknown` + zod). Types come from zod:
  `type EventItem = z.infer<typeof EventSchema>`.
- Styling: CSS classes + CSS variables (light/dark tokens). **No new inline
  `style={{…}}` objects** except truly dynamic values (e.g. a city color).
  When touching an old component, move its inline styles to CSS.
- Accessibility: clickable things are `<button>` or `<a>`, never `<div onClick>`.
  Images have meaningful `alt` (or `alt=""` if decorative). Forms have labels.
- Mobile first: the audience is students on phones with limited data. Test at
  360px width. Lazy-load images (`loading="lazy"`), use the `thumb` size in grids.
- Dates: stored as `YYYY-MM-DD` + `HH:mm` (Africa/Casablanca). Compare dates as
  strings or with a helper in `src/lib/dates.ts` — never `new Date("YYYY-MM-DD")`
  vs `new Date()` (UTC shift bug that hid today's events).
- Never mutate props/state arrays (`[...list].sort(...)`, not `list.sort(...)`).
- Keep components under ~250 lines; split when bigger.

## How to work in this repo

1. Read the relevant section of `docs/ROADMAP.md` and `docs/ARCHITECTURE.md`.
2. Plan first: list the files you will create/change and how you will verify.
   Wait for approval on anything that changes the data model, auth, or
   security headers.
3. Implement in small steps. After each step: `npm run lint && npm test && npm run build`.
4. Verify behavior, not just compilation: run `vercel dev` and exercise the
   feature (public page + admin flow) at 360px and desktop widths.
5. Update docs when behavior changes (ARCHITECTURE for API/data, SECURITY for
   anything auth-related, `.env.example` for new env vars).
6. Finish with a short summary: what changed, how it was tested, anything the
   humans must do by hand (Google Console, Vercel env vars).

Use `/next-task` to pick up the roadmap and `/audit` before opening a PR.

## Things the humans must do (Claude cannot)

- Google Cloud Console setup, OAuth consent publishing, refresh token → `docs/GOOGLE_SETUP.md`
- Adding env vars in Vercel (Production + Preview)
- Getting written consent from members whose name/phone appear on the site
- Deciding content: which cities, which bureau members, guide texts
