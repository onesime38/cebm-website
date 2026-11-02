---
name: next-task
description: Pick up the next unchecked task from docs/ROADMAP.md, plan it, implement it, verify it, and tick it off.
disable-model-invocation: true
argument-hint: "[optional task id, e.g. 1.3]"
---

Work on the CEBM roadmap.

1. Read `docs/ROADMAP.md`. If `$ARGUMENTS` names a task id, take that one;
   otherwise take the first unchecked `- [ ]` task whose dependencies are done.
   Tell the human which task you picked.
2. Read the sections of `docs/ARCHITECTURE.md` and `docs/SECURITY.md` the task
   touches, and the existing code you will change.
3. Check `git status`. If there are uncommitted changes, stop and ask.
   Create a branch `phase-<n>/<short-name>` from the current `main`.
4. Write a plan: files to add/change, schemas or Sheet columns affected,
   env vars needed, how you will verify each acceptance criterion.
   **Stop and wait for approval** if the plan changes the data model, auth,
   roles, security headers, or adds a dependency. Otherwise continue.
5. Implement in small steps. After each step run
   `npm run lint && npm test && npm run build` and fix failures before moving on.
6. Verify every acceptance criterion of the task. For UI, run `vercel dev`
   (or `npm run dev` for front-only work) and check at 360px and desktop width,
   light and dark theme. For API, test the unhappy paths too (401, 403, 400, 409).
7. Update docs that changed (`ARCHITECTURE.md`, `SECURITY.md`, `.env.example`)
   and tick the task's checkbox in `docs/ROADMAP.md`.
8. Propose commit(s) in French Conventional Commits style. Do not push.
9. Report: what changed, how it was verified, and any manual steps for the
   humans (Google Console, Vercel env vars, content to provide).
