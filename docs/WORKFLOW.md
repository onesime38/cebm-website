# Working with Claude Code on CEBM (for the two of us)

## One-time setup

1. Copy this kit into the repo root (`CLAUDE.md`, `.claude/`, `docs/`, `.env.example`)
   and commit: `chore: ajouter la configuration Claude Code et la documentation`.
2. Both of you: pull, then open the Claude desktop app → **Code** tab →
   environment **Local** → project folder → **Open folder…** → pick the
   `cebm-website` folder itself (the one containing `CLAUDE.md`, not a parent
   or a subfolder). Accept the trust prompt if one appears.
   (Terminal users: run `claude` from the repo root — same files apply.)
3. Don't run `/init` — `CLAUDE.md` is already written. If you edit it, keep it
   under ~200 lines; put details in `docs/` or `.claude/rules/`.
4. GitHub: protect `main` (require a pull request + 1 approval), turn on
   Dependabot alerts and secret scanning.

## Splitting the work

Phase 0.1 → 0.2 must be done by **one** person and merged quickly (0.2 moves
every file). After that, work in parallel, for example:

| Person A (backend) | Person B (front-end) |
|---|---|
| 1.1 Google setup (with B) | 0.3 routes, 0.4 head/fonts, 0.5 bugs |
| 1.2 server foundation | 1.4 front part (ContentProvider, loading states) |
| 1.3 scripts | 1.5 login page |
| 1.5–1.7 auth, CRUD, photos API | 1.6–1.7 admin forms, upload UI |
| 1.8 hardening | 1.9 privacy pages, 2.1 guide UI |

Each of you reviews the other's PRs. Pull `main` before starting a task.

## The loop for every task

1. Start a **new session** from the sidebar — one roadmap task per session.
   Permission mode **Manual** or **Accept edits** while you learn the tool
   (avoid Auto at the start).
2. `/next-task 1.2` (or just `/next-task`). Read its plan. Push back if it
   touches files outside the task or adds dependencies you didn't expect.
3. Let it implement. Watch the `+12 -1` diff indicator and review file by
   file; click the stop button as soon as it goes in a wrong direction and tell it why.
4. When it says it's done, test it yourself: ask it to start the dev server
   and check the page in the Browser pane, then on your phone.
5. `/audit`, then the built-in `/code-review` or `/security-review`. Fix findings.
6. Ask Claude to commit (it will ask permission), push the branch, open a PR;
   the other person reviews and merges.

## Prompts that work well

**First session (understanding check, no code):**
> Read CLAUDE.md, docs/ARCHITECTURE.md, docs/ROADMAP.md and src/App.tsx.
> Summarize in 10 lines what the app does today and what we're building.
> List anything in the docs that is unclear or contradictory. Don't change any file.

**Starting a task with constraints:**
> /next-task 0.2 — keep the UI pixel-identical. Do the split in several commits
> (types → lib → components → pages → admin) and run the build after each.

**Asking for proof, not promises:**
> Show me the test that proves a city admin can't edit another city's event,
> and run it.

**When it's stuck in a loop:**
> Stop. Explain in 5 lines what you tried, what failed, and two different
> approaches. Don't write code until I choose.

**Avoid:**
- "Fix everything" / "make the site better" — too broad, it will touch everything.
- Pasting secrets, tokens, or `.env.local` content into the chat.
- Accepting "it should work" — ask it to run the build/tests or show the page.
- Very long sessions: quality drops. Commit, then continue in a new session.

## When you change the rules

- New convention or recurring mistake → add one line to `CLAUDE.md`
  (or the matching `.claude/rules/*.md`).
- Architecture or data change → update `docs/ARCHITECTURE.md` in the same PR.
- New risk → add it to `docs/SECURITY.md` and its review checklist.
