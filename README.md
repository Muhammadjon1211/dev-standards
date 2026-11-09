# NIM Devs — Development Standards

How we design, write, ship and run software. Every new project starts from these documents, and
every existing project is moved toward them when we touch it.

These are written so they can be handed to a teammate **or** an AI coding agent as a spec. When a
rule here and a project's own `AGENTS.md` disagree, the project's `AGENTS.md` wins for that
project, and the deviation should be explained there.

## Layout

| Folder | Document | Covers |
| --- | --- | --- |
| `/` | [ENGINEERING.md](ENGINEERING.md) | Rules that apply to every stack: logic placement, git, env/secrets, data safety, deploy, quality gates |
| `frontend/` | [NEXTJS.md](frontend/NEXTJS.md) | Next.js — the Pages Router blueprint (MUI + SCSS + Apollo) and App Router dashboards |
| `frontend/` | [REACT.md](frontend/REACT.md) | React SPAs on Vite — API client, state, styling tokens, page states, accessibility |
| `frontend/` | [SVELTE.md](frontend/SVELTE.md) | Svelte 5 multi-page apps served by a Python backend (Pi deployments) |
| `frontend/` | [UI_UX.md](frontend/UI_UX.md) | Design and copy rules shared by every frontend |
| `backend/` | [NESTJS.md](backend/NESTJS.md) | NestJS — GraphQL + Mongoose monorepo blueprint, and the REST + Prisma variant |
| `backend/` | [NODEJS.md](backend/NODEJS.md) | Express + TypeScript + Mongoose MVC, plus Node runtime rules |
| `backend/` | [FASTAPI.md](backend/FASTAPI.md) | FastAPI + PostgreSQL (raw SQL), used where the app runs on a Raspberry Pi |
| `ai/` | [LLM_INTEGRATION.md](ai/LLM_INTEGRATION.md) | Putting a model inside a product feature (translation, chat, drafting) |
| `ai/` | [AI_AGENTS.md](ai/AI_AGENTS.md) | Business agents (sales / SMM): human approval, dry run, rules files, dashboards |
| `ai/` | [AI_ASSISTED_DEVELOPMENT.md](ai/AI_ASSISTED_DEVELOPMENT.md) | Working with Claude Code and other agents on our repos |
| `hardware/` | [RASPBERRY_PI.md](hardware/RASPBERRY_PI.md) | Running restaurant systems on Raspberry Pi: OS, services, tunnels, deploy, backups |
| `hardware/` | [ESCPOS_PRINTERS.md](hardware/ESCPOS_PRINTERS.md) | Thermal receipt printers: wiring, udev, print queue, ticket layout |

## Stack at a glance

| Layer | Default choice | When we pick something else |
| --- | --- | --- |
| Web frontend | Next.js + TypeScript | Vite + React for a pure SPA; Svelte 5 for apps served from a Pi |
| API | NestJS + GraphQL (code-first) + MongoDB | NestJS REST + Prisma when Swagger docs are a deliverable; FastAPI + Postgres on a Pi |
| Database | MongoDB Atlas | PostgreSQL when the app runs on-premise or is heavily relational/reporting |
| Auth | JWT, custom guards | Social login (Google / Kakao / Telegram) added through one `socialLogin` path |
| Hosting | Docker Compose on our VPS, one compose project per app | systemd + cloudflared on a Raspberry Pi for in-restaurant systems |
| AI | Claude via the Anthropic API, model name in env | OpenAI-compatible endpoint when the client wants free models |

## The five rules we never break

1. **Business logic lives in the backend.** The frontend renders what the server decided.
2. **Nothing is hard-deleted.** Status flips to `DELETE`, `deletedAt` is stamped, and there is a restore.
3. **No secrets in git.** `.env` is ignored, `.env.example` is committed and documents every variable.
4. **Assume the database is production** until you have proven it isn't. Ask before seeds, migrations and purge scripts.
5. **Every repo has an `AGENTS.md`** that says how to run it, how to check it, and what is surprising about it.
