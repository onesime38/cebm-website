---
paths:
  - "src/**/*.{ts,tsx,css}"
  - "index.html"
  - "public/**"
---

# Front-end rules (src/)

## Data
- All data comes from the API through `src/lib/api.ts` (typed with `shared/`
  schemas). **No `localStorage` for content.** `localStorage` is allowed only
  for the theme preference, wrapped in try/catch.
- Public pages read one cached `GET /api/content` (React context or a small
  hook with stale-while-revalidate). Admin pages call `/api/admin/*` with
  `credentials: "include"` and handle 401 (→ login page), 403, 409, 413, 429.
- Show loading skeletons and friendly French error messages with a retry button.

## Routing
- React Router routes (French slugs): `/`, `/evenements`, `/evenements/:id`,
  `/annonces`, `/galerie`, `/bourse`, `/guide`, `/guide/:slug`, `/contact`,
  `/confidentialite`, `/admin/login`, `/admin/*`.
- Every page sets `document.title` (`"<Page> — CEBM"`).
- Unknown route → a real 404 page, not the home page.
- Event detail must load directly from its URL (refresh / shared link works).

## Admin UI
- Google sign-in button (Google Identity Services). Load the GSI script only
  on `/admin/login`. Send the ID token to `POST /api/auth/google`; never store
  it in localStorage.
- Hide controls the role cannot use, **but assume the server re-checks**.
- Forms: labels, inline validation from the shared zod schema, disable the
  submit button while saving, confirm before archive/delete.
- Image upload: resize in the browser with canvas to `full` (max 1600 px) and
  `thumb` (max 480 px), JPEG/WebP quality ~0.82. Canvas re-encoding strips
  EXIF/GPS — keep it that way. Upload one photo per request (body < 3 MB).

## Images
- Use `<img src="/api/image/{photoId}/thumb">` in lists/grids, `full` only in
  the lightbox / detail view. Always `loading="lazy"` + `decoding="async"` +
  width/height or aspect-ratio to avoid layout shift.

## Styling and accessibility
- CSS classes + tokens in `src/styles/variables.css` (light + dark). No new
  inline style objects except dynamic values. No `<style>` tags inside components.
- Interactive elements are `<button>`/`<a>`/`<Link>`. Visible focus styles.
  Lightbox/dialogs trap focus and close on Escape.
- Contrast ≥ 4.5:1 for text in both themes.
- No inline `<script>` in `index.html` (CSP). The theme bootstrap lives in
  `public/theme-init.js`.

## Copy
- All visible text in French with correct accents and typography
  (« guillemets », espace insécable avant « : ; ! ? » is optional — be consistent).
- Never show sample/fake people or data in production. Seed data lives in
  `scripts/seed.ts` and only goes to the staging Sheet.
