# NestJS Standards

Two shapes, same principles:

| Shape | Used for | Reference repos |
| --- | --- | --- |
| **A. GraphQL monorepo (default)** — code-first GraphQL + plain Mongoose, `-api` + `-batch` apps | Marketplaces, ordering platforms, portfolios | `nestar`, `modu-nest`, `foodbot-nest`, `chayhana-nest`, `modif2`, `nimdevs-nest` |
| **B. REST + Prisma** — Swagger-documented REST under `/api` | Projects where the API docs are a deliverable | `kora/backend` |

The full blueprint for shape A is `NESTJS_GRAPHQL_PROJECT_STRUCTURE.md` in those repos; this
document is the condensed standard.

---

# Part A — GraphQL + Mongoose monorepo

## A1. Stack

| Concern | Choice |
| --- | --- |
| Framework | NestJS 10/11, **monorepo** (`nest-cli.json` → `"monorepo": true`) |
| API | **GraphQL code-first** (`autoSchemaFile: true`), Apollo driver |
| DB | MongoDB via **plain Mongoose schemas** (`new Schema({...})`, no `@Schema()` decorators) |
| Auth | `@nestjs/jwt` + `bcryptjs`, **custom guards, no Passport** |
| Validation | `class-validator` on `@InputType()` DTOs + global `ValidationPipe` |
| Uploads | `graphql-upload` middleware → `./uploads/<entity>/` (Apollo `uploads: false`) |
| Config | `@nestjs/config`, one root `.env` |
| Cron | `@nestjs/schedule` in a **separate `<project>-batch` app** |
| Realtime | `@nestjs/platform-ws` (raw `ws`, not socket.io) |
| Formatting | Prettier: tabs, single quotes, 120 cols. `npx prettier --check "apps/**/*.ts"` must stay clean |

Scripts keep these names: `build`, `format`, `start`, `start:dev`, `start:dev:batch`,
`start:prod`, `start:prod:batch`, `lint`, `test`, `test:e2e`.

## A2. The 12 laws

1. **Three files per feature:** `<feature>.module.ts`, `.resolver.ts`, `.service.ts`. No repository
   layer, no mappers.
2. **Resolvers are thin:** log, guard, read the caller with `@AuthMember()`, convert ids, delegate.
3. **Services hold all business logic.**
4. **One aggregator module** — `ComponentsModule` imports every feature; `AppModule` imports only it.
5. **Cross-feature work goes through exported services**, never another feature's model.
6. **Every list returns `{ list, metaCounter }`** from `$match → $sort → $facet`.
7. **Every list takes one argument** `{ page, limit, sort?, direction?, search }`.
8. **`sort` is whitelisted** with `@IsIn(availableXSorts)` from `libs/config.ts`.
9. **Counters are denormalized** on the parent and maintained by `<entity>StatsEditor` (`$inc`).
   Never `$count` children at read time.
10. **Nothing is hard-deleted** except by an explicit `remove<Entity>ByAdmin`.
11. **No inline error strings** — `Message` enum in `libs/enums/common.enum.ts`.
12. **Every GraphQL enum is registered** right after declaration with `registerEnumType`.

## A3. Folder structure

```
apps/<project>-api/src/
  main.ts  app.module.ts  app.controller.ts  app.service.ts  app.resolver.ts
  components/
    components.module.ts         aggregator only
    auth/                        module + service + decorators/ + guards/ (no resolver)
    member/ product/ board-article/ comment/ follow/ order/ notice/
    like/ view/                  service only — no resolver
  database/database.module.ts
  libs/
    config.ts                    sort whitelists, image helpers, id helpers, $lookup fragments, tuning knobs
    dto/<entity>/<entity>.ts | .input.ts | .update.ts
    enums/<entity>.enum.ts       common.enum.ts = Message + Direction
    types/common.ts              T, StatisticModifier
    interceptor/Logging.interceptor.ts
  schemas/<Entity>.model.ts      PascalCase file, default export
  socket/socket.gateway.ts socket.module.ts
apps/<project>-batch/src/
  main.ts batch.module.ts batch.controller.ts (@Cron only) batch.service.ts (jobs) lib/config.ts
uploads/                         gitignored
```

A small project with no batch app may use the standard `src/` layout with the same internals.

## A4. Naming

| Thing | Convention |
| --- | --- |
| Feature folder | kebab-case, **singular** (`board-article/`) |
| Schema file | `schemas/<Entity>.model.ts` (PascalCase) |
| Output type / list | `Product` / `Products` |
| Inputs | `ProductInput`, `ProductUpdate`, `ProductsInquiry`, `SellerProductsInquiry`, `AllProductsInquiry` |
| Nested search | inquiry initials + `Search`, not exported (`PISearch`) |
| Stats mutator | `productStatsEditor` |
| Model token | **string** `'Product'` in `forFeature` and `@InjectModel('Product')` |

**Operation grammar**

| Operation | Name | Guard |
| --- | --- | --- |
| Create | `create<Entity>` | `RolesGuard` / `AuthGuard` |
| Read one / list | `get<Entity>` / `get<Entities>` | `WithoutGuard` |
| Update own | `update<Entity>` | `AuthGuard` / `RolesGuard` |
| Owner's list | `get<Owner><Entities>` | `RolesGuard` |
| Toggle like | `likeTarget<Entity>` | `AuthGuard` |
| Admin list / update / delete | `getAll<Entities>ByAdmin`, `update<Entity>ByAdmin`, `remove<Entity>ByAdmin` | `RolesGuard(ADMIN)` |

**Every admin endpoint ends in `ByAdmin`** and carries `@Roles(MemberType.ADMIN)` +
`@UseGuards(RolesGuard)`. The suffix makes the privileged surface greppable in one pass. Admin
operations sit at the bottom of the resolver under a `/** ADMIN **/` banner.

## A5. Field naming

Every field is prefixed with the entity name: `memberNick`, `productPrice`, `orderStatus`,
`likeRefId`, `viewGroup`. Unprefixed: `_id`, `createdAt`, `updatedAt`, `deletedAt`, foreign keys.

Standard enums per actor: `<Actor>Type` (must include `ADMIN`), `<Actor>Status`
(`ACTIVE | BLOCK | DELETE`), `<Actor>AuthType` (`PHONE | EMAIL | TELEGRAM | GOOGLE | KAKAO`).
Per resource: `<Resource>Status` (`ACTIVE | <TERMINAL> | DELETE`, plus `HOLD`/`DRAFT` where content
can be unpublished). Polymorphic join groups (`LikeGroup`, `ViewGroup`, `CommentGroup`) share one
member set.

## A6. Schema layer

```ts
const ProductSchema = new Schema(
  {
    productStatus: { type: String, enum: ProductStatus, default: ProductStatus.ACTIVE },
    productTitle:  { type: String, required: true },
    productViews:  { type: Number, default: 0 },          // counters default 0
    memberId:      { type: Schema.Types.ObjectId, required: true, ref: 'Member' },
    deletedAt:     { type: Date },
  },
  { timestamps: true, collection: 'products' },
);
export default ProductSchema;
```

Secrets are `select: false` and read only with an explicit `.select('+memberPassword')`.

## A7. Resolver

```ts
@Roles(MemberType.SELLER)
@UseGuards(RolesGuard)
@Mutation(() => Product)
public async createProduct(@Args('input') input: ProductInput, @AuthMember('_id') memberId: ObjectId): Promise<Product> {
  console.log('Mutation: createProduct');
  input.memberId = memberId;                               // owner is server-assigned
  return this.productService.createProduct(input);
}
```

- A single id argument is a named scalar (`@Args('productId') input: string`), converted with
  `shapeIntoMongoObjectId()` in the resolver.
- Everything else is one `@Args('input')` object.
- On self-update, `delete input._id` and pass the caller id separately.
- `@Roles(...)` goes **above** `@UseGuards(RolesGuard)`.

## A8. Guards

| Guard | Token | Behaviour |
| --- | --- | --- |
| `AuthGuard` | required | throws if missing/invalid; sets `request.body.authMember` |
| `RolesGuard` | required | AuthGuard + role check against `@Roles(...)` |
| `WithoutGuard` | optional | decodes if present, otherwise `null`; never throws |

`@AuthMember('_id')` / `@AuthMember()` read what the guard placed on the request.

## A9. Service patterns

- **Create:** `try { create } catch { log; throw BadRequestException(Message.CREATE_FAILED) }`, then
  bump the owner's counter via the owner's exported service.
- **Login:** `.select('+password')`, check `DELETE`/`BLOCK` status **before** comparing the password.
- **Update:** the filter is the ownership check — `{ _id, memberId, productStatus: ACTIVE }`;
  stamp `soldOutAt` / `deletedAt` server-side; unwind counters on any visibility change;
  `findOneAndUpdate(filter, input, { new: true }).exec()`.
- **Read one:** `.lean()`; for logged-in viewers record the view once, `$inc` the counter, attach
  `meLiked`; fetch the owner with `getMember(null, id)` so it doesn't record a view on the owner.
- **List:**

```ts
const result = await this.productModel.aggregate([
  { $match: match },
  { $sort: sort },
  { $facet: {
      list: [{ $skip: (page - 1) * limit }, { $limit: limit }, lookupAuthMemberLiked(memberId), lookupMember, { $unwind: '$memberData' }],
      metaCounter: [{ $count: 'total' }],
  } },
]).exec();
```

  `$skip/$limit` come first inside `list` so lookups run only on the page. Faceted filters are
  built in a private `shapeMatchQuery(match, input)`. `$lookup` fragments live in `libs/config.ts`.
- Reads `.lean()`, writes `findOneAndUpdate(..., { new: true })`, everything `.exec()`.
- **Compare ids with `isSameId(a, b)`** — a GraphQL arg is a string, the document field is an
  ObjectId, so `===` is always false.
- **Soft delete + restore:** delete flips status to `DELETE` and stamps `deletedAt`; restore brings
  it back to `HOLD` / `DRAFT`, not straight to `ACTIVE`, and rewinds counters.
- Public and admin inquiry DTOs are separate classes: public defaults to `ACTIVE`, admin to every status.
- Multi-document writes (order + items, stock decrement) use a Mongoose session/transaction.
- Per-viewer and derived fields are `@ResolveField`s (`meLiked`, `memberViewerIsFollowing`,
  `productTryOn`) — not computed in the client.

## A10. Errors

1. Services throw Nest exceptions with `Message.*` — and use the **right** one: `NotFoundException`,
   `UnauthorizedException`, `ForbiddenException`, `BadRequestException`, `ConflictException`.
2. Driver errors are caught at the write boundary, logged, translated.
3. `formatError` in `AppModule` flattens everything to `{ code, message }`.
4. **Browse lists return `{ list: [], metaCounter: [] }`** for no rows, not an error.

## A11. Auth tokens

- Sign a **small payload** — `{ _id, memberNick, memberType, memberStatus }` — not the whole member
  document. Load the rest when needed. Re-issue the token when those fields change.
- `SECRET_TOKEN` missing = startup error.
- Social login: one `socialLogin(provider, credential)` mutation. Google ID token verified
  server-side, Kakao code exchanged server-side, Telegram HMAC checked with the bot token.
  `getAuthProviders` returns the public ids that are configured. Social members have no
  phone/password (`memberSocialId` + unique index on `authType + socialId`).
- Self-registration never accepts a `role`.

## A12. Realtime

- All reads and writes stay in GraphQL. The `ws` gateway only **pushes** events (`message`,
  `unread`) to a member's open tabs after the client sends `{ event: 'auth', data: { token } }`.
- Server pings every 30 s.

## A13. Build order for a new project

1. Scaffold, monorepo, `.env`, `.gitignore`, `.prettierrc`.
2. `libs/types/common.ts`, `libs/enums/common.enum.ts`, all enums.
3. `libs/config.ts` (sorts, image helpers, id helpers).
4. All schemas.
5. Database module, AppModule, `main.ts` — **boot and see `sayHello` work** before any feature.
6. Logging interceptor → `auth/` → `member/` (signup, login, checkAuth, update) → `view/` → `like/`
   → lookups → primary resource → favorites/visited → `follow/` → secondary resource → `comment/`
   → uploads → orders → batch → socket.

## A14. Gotchas

- Keep `"incremental": false` in `tsconfig` — with it on, `deleteOutDir` left `dist` with only the
  changed files.
- A stale API process may still hold the port; `nest start --watch` then rebuilds but can't start.
  If new resolvers are missing, check which PID owns the port.
- Atlas latency from the dev machine can exceed short server-side timeouts — let a timeout surface
  as 500, not as a false 404.
- Remove stray editor-inserted imports (`node:vm`, `node:console`).

---

# Part B — REST + Prisma variant

## B1. Stack

NestJS 11, Prisma 6 (MongoDB or Postgres), REST under global prefix `/api`, **Swagger at
`/api/docs`**, stateless JWT, `class-validator`, Jest. Node version pinned and enforced by
`scripts/check-node.cjs`.

## B2. Conventions

- **Controllers stay thin:** DTO-validated input → one service call → return.
- **Every body/query has a DTO** in `<feature>.dto.ts`. Global pipe:
  `whitelist: true, forbidNonWhitelisted: true, transform: true` — an unknown property is a 400.
  Query numbers use `@Type(() => Number)`.
- **Auth on a route:** `@ApiBearerAuth() @UseGuards(JwtAuthGuard)` + `@CurrentUser() user`. Admin adds
  `RolesGuard` + `@Roles(Role.ADMIN)` — `RolesGuard` alone does not authenticate.
- **Swagger annotations are mandatory:** `@ApiTags`, `@ApiBearerAuth`, `@ApiConsumes('multipart/form-data')`.
- **Ownership in the service:** `getOne(userId, id, isAdmin)` throws `ForbiddenException`. Admin
  paths pass `isAdmin = true` rather than skipping the check. Never trust a client-supplied user id.
- Enums come from `@prisma/client`, never re-declared.
- **Every support correction takes a reason** and writes an audit row ("who changed this, and why").
- Impersonation ("view as") tokens are **read-only by HTTP method** in the JWT strategy, short-lived,
  and audited.
- Background timers `unref()` their handle so tests and scripts can exit; idempotency comes from a
  stamped column (`lastDigestAt`), not the schedule.
- Rate limiting: global throttler + tighter limits on sign-in, registration, password reset, contact form.
- Suspension is checked per request in the JWT strategy, so it takes effect immediately.

## B3. Testing

- Jest + `ts-jest`, `*.spec.ts` next to the service.
- Instantiate the service directly with a hand-rolled Prisma mock; assert the returned shape or the
  thrown exception type. **No tests that hit the real database.**
- `scripts/smoke-api.mjs` walks every role against a running localhost server, including the
  negatives (student → admin route = 403, tenant A reading tenant B = 404). It refuses non-localhost
  and requires `--write-local-data`.

## B4. Data safety

`DATABASE_URL` often points at a live cluster. `db:push`, `seed`, purge and import scripts need the
owner's confirmation every time.
