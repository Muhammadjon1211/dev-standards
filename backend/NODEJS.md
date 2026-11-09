# Node.js (Express) Standards

Layered MVC-style **Express + TypeScript + Mongoose** backend that can serve two surfaces from one
codebase: a JSON API for the SPA and a server-rendered EJS admin. Reference repos: `kudo/Kudo`,
`burak`. The full LLM-ready blueprint is `BACKEND-STRUCTURE-PROMPT.md` in those repos.

The second half of this document is runtime rules for **any** Node service (Express, Nest, bots, agents).

---

## 1. Stack

- Node, TypeScript `strict`, Express 4, Mongoose.
- Auth: JWT in an **httpOnly** cookie for the API; `express-session` + `connect-mongodb-session` for the EJS admin.
- `bcryptjs`, `multer` + `uuid` uploads, `morgan`, `cors`, `cookie-parser`, `dotenv`, `ejs`.
- Scripts: `start`, `start:dev`, `build` (`tsc`).

## 2. Directory layout

```
src/
  server.ts            dotenv → mongoose.connect → app.listen (inside .then)
  app.ts               middleware, sessions, views, routers — in commented sections
  router.ts            API routes (JSON)
  router-admin.ts      SSR admin routes (EJS)
  controllers/<entity>.controller.ts
  models/<Entity>.service.ts      SERVICE layer (business logic)
  schema/<Entity>.model.ts        Mongoose schemas
  libs/
    config.ts          constants + helpers (AUTH_TIMER, MORGAN_FORMAT, shapeIntoMongooseObjectId)
    Errors.ts          Errors class + HttpCode + Message
    types/common.ts    T
    types/<entity>.ts  Entity / EntityInput / EntityUpdateInput / EntityInquiry
    enums/<entity>.enum.ts
    utils/uploader.ts  multer factory keyed by folder
  views/  public/
uploads/<entity>/
```

**`models/` holds `*.service.ts`, `schema/` holds `*.model.ts`.** The split is intentional; keep it.

## 3. Bootstrap

- `server.ts` is the only file that reads `.env` and opens a port. Nothing listens until Mongo connects.
- `app.ts` sections, in order: `/* 1-ENTRANCE */` (static, uploads, body parsers, cors, cookies,
  morgan) → `/* 2-SESSIONS */` → `/* 3-VIEWS */` → `/* 4-ROUTES */` (`/admin` first, `/` last).
- Add a **404 handler and a global error handler** as a backstop after the routers.

## 4. Routers

- Two flat routers, no per-feature route files. They are lookup tables:
  `method + path → [middleware…] → controller.handler`, grouped by `/** Entity **/` comments.
- Paths: `/<entity>/<action>` — `all`, `create`, `update`, `detail`, `:id`.
- **Literal paths before `/:id`** siblings.
- Auth middleware per route, never global.

## 5. Controllers

```ts
memberController.getMemberDetail = async (req: ExtendedRequest, res: Response) => {
	try {
		console.log('getMemberDetail');
		const result = await memberService.getMemberDetail(req.member);
		res.status(HttpCode.OK).json(result);
	} catch (err) {
		console.log('Error, getMemberDetail', err);
		if (err instanceof Errors) res.status(err.code).json(err);
		else res.status(Errors.standard.code).json(Errors.standard);
	}
};
```

- Pull from the request, coerce primitives, attach upload paths, call **one** service method, respond.
- **Validate at the boundary:** reject `NaN` page/limit, unknown enum values, empty bodies.
- No DB access, no bcrypt/jwt, no business rules in controllers.
- Type handlers (`RequestHandler`) even when the controller is an object literal.
- SSR handlers render or redirect; write the session then redirect **inside** `req.session.save()`.
  Never interpolate values into `<script>alert(...)</script>` — render an error view.

## 6. Services

- One class per entity, default export, models and peer services as `private readonly` fields.
- Public methods `async` with explicit return types.
- Grouped `/** SPA */` and `/** SSR */`; SSR variants get distinct names (`login` / `processLogin`).
- **The service is the only layer that touches a model.**
- Failure = `throw new Errors(HttpCode.X, Message.Y)`. Never return `null` to mean failure.
- Owner-scoped writes use `findOneAndUpdate({ _id, memberId }, …)` — **never**
  `findByIdAndUpdate({ … })` (the filter is treated as an id and the ownership check is silently lost).
- Null-check a query result **before** touching it.
- Multi-document writes (order + items) in a transaction, or clean up on failure.
- Lists: `$match → $sort → $skip → $limit → $lookup…`; joined fields declared on the type under
  `/* from aggregations */`.

## 7. Schemas

Entity-prefixed fields, `{ timestamps: true }`, explicit `collection` where `$lookup` needs it,
`ref` on relations, `select: false` on secrets, `index: { unique: true, sparse: true }` for uniqueness.
TS enums drive schema enums so values have one source of truth.

## 8. Errors

```ts
export enum HttpCode { OK = 200, CREATED = 201, BAD_REQUEST = 400, UNAUTHORIZED = 401,
	FORBIDDEN = 403, NOT_FOUND = 404, CONFLICT = 409, INTERNAL_SERVER_ERROR = 500 }
```

All messages are `Message` members. Use `BAD_REQUEST` / `CONFLICT` for failed writes — not 304.

## 9. Auth

- API: `verifyAuth` (required) and `retrieveAuth` (optional, always `next()`).
- Cookie: `httpOnly: true`, `secure` + `sameSite` in production; `maxAge` from `AUTH_TIMER`.
- Sessions: `saveUninitialized: false`.

## 10. Code style

- Prettier settings follow the repo. Old Express repos use 4 spaces + double quotes; **new repos use
  the house style (tabs, single quotes, 120 cols)**. Don't mix within a repo.
- `console.log('<handlerName>')` at the top of each handler, `console.log('Error, <handlerName>', err)` in each catch.
- Build one entity end-to-end (enum → types → schema → service → controller → routes → messages)
  before starting the next; commit per entity.

---

# Node runtime rules (every Node service)

- **Pin Node** (`.nvmrc` / `.node-version` / `engines`) and fail fast with a `check-node.cjs` guard.
  Current LTS for new projects.
- **Dev runner:** use `tsx`. `ts-node-dev` / `ts-node --watch` break when a parent folder's
  `package.json` has `"type": "module"`. For long-polling bots, run plain `tsx` (not `tsx watch`) —
  a watcher restart starts a second poller and Telegram answers 409. File watchers on Windows can
  miss saves: restart manually when a route change doesn't show up.
- **Respond first, then notify.** `res.json(...)` → then fire-and-forget the Telegram / email call
  (`notifyUser(...).catch(log)`). Never await a third party before responding.
- **Compute every value before `res.json`.** A `const` declared after the `res.json({ x })` line is a
  temporal-dead-zone ReferenceError.
- **BigInt** (Prisma ids, Telegram chat ids) can't be `JSON.stringify`'d — send `String(id)`.
- Falsy ids: `0` is a valid value. Check `=== undefined || === null`, not `!id`.
- Outbound HTTP always has a timeout.
- On Windows, `prisma generate` fails with EPERM while the server holds the engine DLL — stop the
  server first.
- One package manager and one lockfile per repo.
