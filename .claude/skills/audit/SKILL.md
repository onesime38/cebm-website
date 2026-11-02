---
name: audit
description: Review the current branch's changes against the CEBM security checklist and quality rules before a pull request.
disable-model-invocation: true
context: fork
argument-hint: "[base branch, default main]"
---

You are reviewing someone else's change with fresh eyes. Be strict and concrete.

1. Base branch = `$ARGUMENTS` or `main`. Run `git diff <base>...HEAD --stat`
   and read the full diff. Read `docs/SECURITY.md` and `CLAUDE.md`.
2. Check every item of the "Review checklist" in `docs/SECURITY.md` against
   the diff. Also check:
   - any new `VITE_` variable, hardcoded secret, ID, email, or phone number
   - admin route without `requireRole`, or city scoping enforced only in the UI
   - zod validation missing or not `.strict()`; body size not capped
   - Sheets write not using `RAW`; Drive call with a scope other than `drive.file`
   - image route accepting arbitrary Drive IDs
   - `dangerouslySetInnerHTML`, unvalidated `href`/`src`, `target="_blank"` without `rel="noopener noreferrer"`
   - personal data newly exposed on a public endpoint (`/api/content`)
   - error responses leaking stack traces or Google error bodies
   - `localStorage` used for content; props/state mutated in place
   - `<div onClick>`, images without `alt`, inline styles added
   - CSP in `vercel.json` loosened (`unsafe-eval`, wildcard hosts)
3. Run `npm run lint`, `npm test`, `npm run build`, `npm audit`.
4. Report findings ranked by severity (Critical / High / Medium / Low) with
   file:line, the concrete failure scenario, and the fix. If nothing is wrong
   in a category, say so in one line. End with "Ready for PR: yes/no".

Do not modify files during the audit.
