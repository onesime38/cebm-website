# Google + Vercel setup (done by a human, once)

Time: about 45 minutes. Do it with the CEBM Google account, not a personal one.
Google renames console menus from time to time; the names below are the
current ones ("Google Auth Platform"), older guides call it "OAuth consent screen".

## 1. The CEBM Google account

- Use (or create) an account owned by the association, e.g. `cebm.maroc@gmail.com`.
- Turn on 2-step verification. Add recovery options that **two** bureau members control.
- This account will own the Sheet and the photos. Free storage: 15 GB, shared with its Gmail.

## 2. Google Cloud project

1. Go to https://console.cloud.google.com while signed in as the CEBM account.
2. Create a project named `cebm-site`.
3. APIs & Services → Library → enable **Google Sheets API** and **Google Drive API**.

## 3. Google Auth Platform (consent screen)

1. Google Auth Platform → **Branding**: app name `CEBM Site`, support email = CEBM account,
   developer contact = CEBM account. Skip the logo for now (a logo triggers a
   brand review). Add the production domain under authorized domains once you have it.
2. **Audience**: user type **External**.
3. **Data access** → add scopes: `openid`, `.../auth/userinfo.email`,
   `.../auth/userinfo.profile`, `.../auth/drive.file`.
   Do **not** add `drive` or `spreadsheets` — those are sensitive/restricted and
   require Google's verification. `drive.file` is enough because the app
   creates the Sheet and the photos itself.
4. **Audience → Publish app** so the status is **In production**.
   ⚠️ If the app stays in *Testing*, the refresh token expires after 7 days and
   the site stops working every week.

## 4. OAuth client

Google Auth Platform → **Clients** → Create client → **Web application**, name `cebm-web`.

- Authorized JavaScript origins:
  - `https://<your production domain>` (and `https://<project>.vercel.app`)
  - `http://localhost`
  - `http://localhost:3000`
- Authorized redirect URIs:
  - `http://localhost:5555/oauth2callback` (used only by the refresh-token script)

Copy the **Client ID** and **Client secret**.

## 5. Refresh token (server access to the CEBM Drive)

After Claude Code has written `scripts/get-refresh-token.ts` (roadmap 1.3):

```bash
# .env.local must contain GOOGLE_CLIENT_ID and GOOGLE_CLIENT_SECRET
npx tsx scripts/get-refresh-token.ts
```

A browser opens → sign in **as the CEBM account** → allow. The script prints a
refresh token once. Put it in `.env.local` as `GOOGLE_REFRESH_TOKEN` and save
a copy in the association's password manager. Never paste it in chat, issues,
or commits.

## 6. Create the Sheets and folders

```bash
npx tsx scripts/bootstrap-google.ts --env staging
npx tsx scripts/bootstrap-google.ts --env production
```

Each prints `SHEET_ID` and `DRIVE_PHOTOS_FOLDER_ID`. Then:

- Open each Sheet in Drive → Share → add only bureau members who may edit
  content by hand (editors). The script protects the `admins` and `audit_log`
  tabs so only the CEBM account can change them; editors can change the
  content tabs.
- Seed staging only: `npx tsx scripts/seed.ts`.

## 7. Secrets

```bash
openssl rand -base64 48   # run twice: one SESSION_SECRET for staging, one for production
```

## 8. Vercel

Import the GitHub repo in Vercel (framework preset: Vite). Settings → Environment Variables:

| Variable | Production | Preview + Development |
|---|---|---|
| `GOOGLE_CLIENT_ID` | client id | same |
| `VITE_GOOGLE_CLIENT_ID` | client id | same |
| `GOOGLE_CLIENT_SECRET` | secret | same |
| `GOOGLE_REFRESH_TOKEN` | token | same |
| `SHEET_ID` | production sheet | staging sheet |
| `DRIVE_PHOTOS_FOLDER_ID` | production folder | staging folder |
| `SESSION_SECRET` | secret #1 | secret #2 |
| `BOOTSTRAP_ADMIN_EMAILS` | your two Google emails | same |
| `SITE_ORIGIN` | `https://<production domain>` | `http://localhost:3000` |

Locally: `vercel link`, then create `.env.local` by hand from `.env.example`
with the staging values (Claude Code is not allowed to read it).

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `invalid_grant` in logs | Refresh token revoked or app was in Testing → publish the app, re-run step 5, update Vercel, redeploy. |
| Google button: "origin not allowed" | Add the exact origin (scheme + host + port) in step 4. Takes a few minutes to apply. |
| `File not found` for a photo added in Drive by hand | Expected with `drive.file`: upload photos through the admin panel. |
| 403 after sign-in | Email not in `admins` (active) nor in `BOOTSTRAP_ADMIN_EMAILS`. |
