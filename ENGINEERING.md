# Engineering Standards (all stacks)

Rules that apply to every project, whatever the language. Stack-specific documents build on this one.

---

## 1. Where logic lives

**If a rule could live in either the frontend or the backend, it lives in the backend.**

- Validation, authorization, eligibility rules, prices, totals, fees, derived values, counters and
  state transitions belong in services / resolvers / DTO validators.
- A rule enforced only in the client is not enforced at all — anyone can call the API directly.
  A rule enforced in both places is two copies that drift apart.
- A client-side check is allowed **only** as a convenience on top of a real server rule (hiding a
  button the API would reject anyway). Mark it with a comment saying so.
- Prefer server-derived fields over client computation: GraphQL `@ResolveField`s such as
  `memberViewerIsFollowing`, or a REST response that already contains `missingDocuments`.
- Quotes are display-only. If the client asks "what will delivery cost?", the server answers, and
  then measures and prices **again** when the order is created. Never trust a total sent by the client.
- Ids that identify *the caller* come from the token, never from the request body.

Real bugs that came from breaking this rule: a delivery-coverage check computed client-side from a
fallback list with no coordinates (it always passed); an admin console guarded only by a client
component.

## 2. Data safety

- **Soft delete everywhere.** Status enum carries `DELETE` (or `hidden` / `active=false`),
  `deletedAt` is stamped by the server. Every remove has a matching restore, and a restore comes
  back to a *non-live* state (`HOLD`, `DRAFT`) rather than straight onto the site.
- Orders, payments and history rows are never deleted.
- **Dev and prod are often the same database** (shared Atlas cluster). Until you have checked the
  connection string, treat every write as production. Confirm with the project owner before
  running `db:push`, seeds, migrations, imports or purge scripts.
- Migrations are **numbered, append-only and idempotent** (`IF NOT EXISTS`). Never edit an applied
  migration. Keep a migration that drops columns unexecuted until a rollback is no longer needed.
- Before any risky change on a live system: take a backup (`pg_dump` / `mongodump` / copy of the
  file with a timestamped `.bak-<reason>-<yyyymmdd-hhmmss>` suffix) and write down the rollback.
- Test scripts that create rows must refuse non-localhost targets, require an explicit flag
  (`--yes`, `--write-local-data`), name their rows recognisably (`Smoke …`, `smoke.*@example.test`)
  and clean up (soft-delete) after themselves.

## 3. Secrets and environment

- `.env`, `.env.local`, `.env.development` are **gitignored**. `.env.example` is committed and lists
  every variable with a one-line comment. Adding a setting = adding it to `.env.example` in the same commit.
- All env reads go through **one config module** (`libs/config.ts`, `app/config.py`). No
  `process.env.X` scattered through the code.
- Anything exposed to the browser (`NEXT_PUBLIC_*`, `VITE_*`, `REACT_APP_*`) is **public by
  definition**. Never put a key there. Third-party keys a browser widget needs (Google client id,
  Kakao JS key) are served from the backend config endpoint so a change does not need a rebuild.
- A secret that was ever committed or pushed is **compromised**: rotate it, then untrack it.
  Removing it from history needs a rewrite + force push and an explicit decision.
- Optional integrations **degrade, they don't crash**: an unset `GOOGLE_CLIENT_ID` hides the Google
  button and the endpoint answers `503`; a missing `ANTHROPIC_API_KEY` turns the translation job off.
- Weak defaults (`admin123`) are refused at startup in production.

## 4. Naming

- **Entity-prefixed fields** in every data model: `memberNick`, `productPrice`, `orderStatus`,
  `likeRefId`. Only `_id`, `createdAt`, `updatedAt`, `deletedAt` and foreign keys (`memberId`) are
  unprefixed. This is the most visible convention in our codebases — keep it.
- Enums: `PascalCase` name, `SCREAMING_SNAKE` members whose value equals the name.
- Never render a raw enum in the UI. Map it through a label helper.
- One language for code comments and identifiers: **English**. UI copy is translated through i18n files.
- Watch spelling in shared identifiers. A typo in a schema field or enum member (`orderStaus`,
  `craetedAt`, `MemberStatus` vs `memberStatus` in a match object) becomes a silent data bug.

## 5. Errors

- Every user-facing message is a constant in one catalogue (`Message` enum, or a `snake_code`
  error key the frontend translates). No inline string literals in a throw.
- Use the right status: 400 validation, 401 not signed in, 403 not allowed, 404 not found, 409
  conflict. Do not throw 500 for expected domain failures.
- Raw driver/DB errors are logged server-side and never leak to the client.
- **No silent `catch {}`** anywhere. A failed action shows a visible error.
- Notifications to third parties (Telegram, email) are **fire-and-forget after the response** is
  sent, never awaited before it, and their failures are logged.

## 6. Git

- Branches: `main` (or `master`) = production; `develop` = integration when the project uses it.
  Feature branches are short-lived. One default branch that a fresh `git clone` gets is the
  deployable one — never let it fall behind.
- **Conventional Commits:** `feat: …`, `fix: …`, `refactor: …`, `chore: …`, `docs: …`.
  Projects using `standard-version` generate `CHANGELOG.md` from them.
- Commit as yourself. **No AI attribution** — no `Co-Authored-By: Claude`, no "Generated with
  Claude Code" lines in commits or PR descriptions.
- One lockfile per project, matching its package manager (`yarn.lock` *or* `package-lock.json`
  *or* `uv.lock`). Never create a second one.
- Do not reformat lines you are not changing. Formatting-only changes go in their own commit.
- Never commit build output, `uploads/`, `node_modules/`, `.env*` (except `.env.example`), tunnel
  credentials or database dumps.

## 7. Quality gates

Every repo defines the commands that must pass before a push, and lists them in `AGENTS.md`:

| Gate | Typical command |
| --- | --- |
| Type check / build | `npm run build`, `tsc -b`, `svelte-check --fail-on-warnings` |
| Lint / format | `npx prettier --check "apps/**/*.ts"`, `uv run ruff check app scripts` |
| Unit tests | `npm test` (Jest), `uv run pytest` |
| Smoke test | `scripts/smoke-api.mjs`, `python -m scripts.smoke_api` against localhost |
| E2E | `yarn test:e2e` (Playwright) when the project has it |

A change to an endpoint is not done until the smoke test passes against a running server.
Do not leave dev servers running after a check — they hold ports and confuse the next session.

## 8. Deploy

- **VPS:** each app is its own Docker Compose project with an explicit `name:` line (so one app's
  `down` cannot clobber another), bound to `127.0.0.1:<port>` or a dedicated public port, built on
  the server. Each repo ships a `deploy/redeploy.sh`. Start services **by name**
  (`docker compose up -d api dashboard`).
- **Raspberry Pi:** systemd unit generated by an install script + cloudflared tunnel. See
  [hardware/RASPBERRY_PI.md](hardware/RASPBERRY_PI.md).
- Private repos are shipped as a `git archive` tarball, or the machine gets its **own** deploy key.
  Never embed a personal access token in a remote URL.
- Every deploy keeps the previous build (`dist.prev`, `app.prev`, image tag) so rollback is one move.
- Frontend env vars that are baked at build time (`NEXT_PUBLIC_*`, `VITE_*`) need a rebuild, not a
  restart.
- After deploying, verify on the real URL, not just the build log.

## 9. Documentation every repo must have

- `README.md` — what it is, how to run it locally (copy-pasteable commands), required env.
- `AGENTS.md` — conventions, gates, gotchas, known gaps "do not report as bugs". Read it before
  editing; update it in the same change when a convention changes.
- `.env.example` — every variable, commented.
- Optional: a blueprint document (`DEVELOPMENT_STANDARDS.md`,
  `NESTJS_GRAPHQL_PROJECT_STRUCTURE.md`) when the repo is meant to be cloned for new projects.

## 10. Scope

- Build what was asked. If the request is "only the staff interface", do not migrate the whole
  system. Ask before reusing or deleting anything that was over-scoped.
- One reusable codebase per product family (e.g. the restaurant blueprint): per-client copies get
  their own repo and env. Never put a real client's keys in the blueprint.
