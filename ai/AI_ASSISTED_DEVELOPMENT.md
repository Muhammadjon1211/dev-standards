# AI-Assisted Development Standards

How we work with Claude Code (and any other coding agent) on our repositories.

---

## 1. Every repo is agent-ready

- `AGENTS.md` at the root of each repo (and in each sub-repo, e.g. `backend/`, `frontend/`). It contains:
  - **What this is** — one paragraph, runtime, port, database, who the clients are.
  - **Commands** — install, dev, build, lint, test, smoke. Copy-pasteable.
  - **Layout** — the folder map.
  - **Conventions** — the rules this repo enforces (link back to this standards repo).
  - **Gotchas** — what has bitten us before.
  - **Known gaps — do not report as bugs.**
- `CLAUDE.md` may simply point to `AGENTS.md`.
- **Read `AGENTS.md` before editing.** Update it in the same change when a convention or a
  user-facing decision changes (e.g. "user rejected the small + button").
- Blueprint repos carry a spec an agent can scaffold from (`DEVELOPMENT_STANDARDS.md`,
  `NESTJS_GRAPHQL_PROJECT_STRUCTURE.md`, `BACKEND-STRUCTURE-PROMPT.md`).

## 2. How we give tasks

- Say what is in scope and what is not. "Only the staff interface" means only that.
- Point at the nearest existing sibling ("make it like `createProduct`"). The agent copies its shape.
- Give screenshots / reference folders for UI work.
- For physical output (receipts, devices), send a photo.

## 3. What the agent must do

- Match the repo's conventions over its own defaults (naming, file layout, formatting).
- Run the repo's gates (build, lint, tests, smoke) and report the real result — failures included.
- Put business logic in the backend (see [ENGINEERING.md](../ENGINEERING.md)).
- Ask before anything hard to reverse: schema pushes, seeds, purges, force pushes, deleting files
  or databases, deploying to a live device, sending messages to real customers.
- Take a backup before changing a live machine and state the rollback.
- Stop dev servers it started.
- Keep the change minimal — no drive-by refactors or reformatting.

## 4. What the agent must not do

- **No AI attribution in git.** No `Co-Authored-By: Claude`, no "Generated with Claude Code" in
  commits or PRs. Commit as the developer's own git identity.
- Never print, copy or commit secrets. Never put credentials in a standards doc, memory note or README.
- Never run test scripts that write data against a non-local database without confirmation.
- Never re-add features the client removed.

## 5. Windows machine gotchas

- **PowerShell 5.1 corrupts UTF-8.** `Get-Content` reads BOM-less UTF-8 as ANSI and `Set-Content`
  writes a BOM — round-tripping a `.ts` file turns `·`, `…`, `↑` into mojibake. Edit with the
  editor/Edit tool or Node `fs`; if PowerShell is unavoidable use
  `[IO.File]::ReadAllText/WriteAllText` with `[Text.UTF8Encoding]::new($false)`.
- `git core.autocrlf=true` means Windows trees are CRLF. When comparing with a Linux deploy,
  strip `\r` first.
- `C:\Users\<user>\package.json` with `"type": "module"` breaks `ts-node-dev`. Use `tsx`.
- Native `ssh` can't take a password non-interactively; use Posh-SSH or keys.

## 6. Memory and handover

- Decisions that aren't visible in code (why a feature was removed, which DB is really prod, which
  tunnel serves which domain) go into the repo's `AGENTS.md` or the project notes — not just chat.
- Write dates as absolute dates.
- Record what is deployed vs. what is only committed vs. what is only local.
