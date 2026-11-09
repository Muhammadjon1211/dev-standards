# FastAPI Standards

Python backends — chosen when the app runs **on a Raspberry Pi in the restaurant** (QR menu +
kitchen printing) or when the project is an AI agent. Reference repos: `navai-back`,
`silkroad-new/backend`, `sales-agent`, `smm-agent`.

---

## 1. Stack

| Concern | Choice |
| --- | --- |
| Runtime | Python 3.11+, managed by **`uv`** (`uv.lock` is the source of truth) |
| Web | FastAPI |
| DB | PostgreSQL via **psycopg 3 async pool**. **No ORM** — SQL lives in services/routers |
| Lint | `ruff` — `uv run ruff check app scripts` must stay clean |
| Tests | `pytest` + a localhost smoke script |
| Realtime | Server-Sent Events (admin live view), Postgres `LISTEN/NOTIFY` for triggers |
| Serving | The backend serves the built frontend `dist/` at `/` |

Agents / small tools may use SQLite in `data/`.

## 2. Layout

```
app/
  __main__.py          `python -m app --port 5000` (Windows-safe event loop)
  main.py              app factory, lifespan (start background workers here)
  config.py            the ONLY place env is read
  routers/             one file per area; admin routers carry Depends(require_admin)
  services/            business logic, one module per concern
  printing/            device code (see hardware/ESCPOS_PRINTERS.md)
db/
  migrations/NNN_name.sql   numbered, append-only, idempotent
  setup.sh                  creates role/db, applies migrations, records schema_migrations
scripts/smoke_api.py
deploy/                     install_service.sh, backup/, vps/
```

## 3. Conventions

- **Logic lives here, not in the frontend.** Prices, totals, delivery fees, pair rules, report
  aggregation (in SQL) are server-side.
- **JSON is camelCase.** Do it with SQL aliases (`daily_number AS "dailyNumber"`) and keep one
  definition per shape (`ORDER_COLUMNS`, `PRODUCT_COLUMNS`). Update the frontend `types.ts` in the
  same change.
- **Errors:** `HTTPException(status, detail={"error": "snake_code", ...})`. The frontend switches on
  `detail.error`. Never raise a bare string.
- **Config only through `app/config.py`.** Every new setting also goes into `.env.example` with a comment.
- **Runtime settings the owner edits** (name, opening hours, order types, delivery pricing) live in
  one JSON row (`app_settings`), loaded into memory at startup and swapped after each save. `.env`
  only seeds them on first run. Read through `settings.current()`, never the raw env values.
- **Nothing is hard-deleted:** products → `status='hidden'`, categories → `deleted`, tables →
  `active=FALSE`. Orders keep history.
- **Admin routes** under `/api/admin/*` with `Depends(require_admin)`. Customer routes are gated by
  sessions (table session or customer token).
- **Route order:** literal paths before `/{id}` siblings (`/orders/stream` before `/orders/{id}`).
- **Allow-lists for nested JSON** (`clean_variants`): a new field must be added there or it is
  silently dropped.
- **Caches:** every write to cached data calls its `invalidate()`.
- **Blocking I/O** (USB printing, heavy CPU) goes through `run_in_executor`.
- **Background workers** (print queue, auto-done, translation job) are started in the lifespan,
  are idempotent, and log failures without crashing the app.
- **Timezone:** business days and printed times use `RESTAURANT_TZ`; keep the DB server timezone
  aligned with it.

## 4. Auth

- Customer accounts: phone + password (scrypt), Google (ID token verified), Telegram (Login Widget /
  Mini App `initData` HMAC). All mint the same opaque token, stored **only as a sha256** in a
  sessions table, sliding expiry.
- Login guard: N wrong passwords per phone → temporary lock.
- Changing a password signs out every other session.
- Signed URLs for physical QR codes (`?t=<id>&k=<key>`): the key is an HMAC or a rotated per-table
  key; the check is never off. If the secret is unset, generate one at first start and store it.
- Third-party ids that the browser needs are returned by `/api/config`; hide the feature when unset.

## 5. External services

- Every call has a timeout and a fallback: if the road-distance API fails, use straight line × 1.3;
  if geocoding is down, take the order at the minimum fee rather than refusing it.
- A **quote endpoint is display-only**; order creation measures and prices again from server data.
- Network / 429 / 5xx → keep the job pending and retry later. Bad key → pause the job, log loudly.

## 6. Migrations

- `db/migrations/NNN_name.sql`, never edit an applied file, `IF NOT EXISTS` everywhere.
- `db/setup.sh` parses `DATABASE_URL`, applies every migration in order, tracks them in
  `schema_migrations` — so a fresh install gets the full schema.

## 7. Testing

- `scripts/smoke_api.py --yes --base http://127.0.0.1:5000` hits every endpoint, creates real rows,
  soft-deletes them at the end, restores any setting it changed (`atexit`), and refuses non-localhost.
- Run it after changing any endpoint.
- On Windows run Python with `PYTHONUTF8=1`.

## 8. Agents and tools in Python

- `pyproject.toml` + `Makefile` targets (`make venv`, `make run`, `make api`, `make dashboard`).
- Start in **dry run** by default; real sending is an explicit setting.
- A read-only dashboard API sits beside the agent on `127.0.0.1` with a bearer token.
- Business actions live in one module (`post_actions`) shared by every channel (Telegram buttons,
  dashboard). See [ai/AI_AGENTS.md](../ai/AI_AGENTS.md).
