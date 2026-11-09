# Next.js Standards

We run two kinds of Next.js project:

| Kind | Router | Used for | Reference repos |
| --- | --- | --- | --- |
| **Product site (blueprint)** | **Pages Router** | Public marketplaces / portfolios talking to our NestJS GraphQL API | `nestar-next`, `modu-next`, `nimdevs-next` |
| **Dashboard / internal tool** | App Router | Owner dashboards in front of a Python or Nest API | `sales-agent/dashboard`, `smm-agent/dashboard` |

Part A is the default for any new customer-facing site. Part B covers dashboards.

---

# Part A — The Pages Router blueprint

## A1. Stack

| Layer | Choice | Notes |
| --- | --- | --- |
| Framework | Next.js 14 — **Pages Router** | `pages/`, `_app.tsx`, `_document.tsx`. Not App Router. |
| Language | TypeScript, `strict: true` | |
| UI kit | **MUI v5** | Layout from `Stack` / `Box` / `Typography` |
| Styling | **Global SCSS** + MUI theme override | No CSS Modules, no Tailwind, no styled-components for layout |
| Data | **Apollo Client 3 + GraphQL** | Pin `@apollo/client ~3.13` (3.14 floods deprecation warnings for `onCompleted`) |
| Global state | **Apollo reactive variables (`makeVar`)** | No Redux, no Zustand, no Context for app state |
| Auth | JWT in `localStorage`, decoded with `jwt-decode` | Attached by an Apollo auth link |
| Realtime | WebSocket link (GraphQL subscriptions or raw `ws`) | Chat, notifications |
| i18n | **next-i18next**, locales `en`, `kr`, `ru` | Keys are the English sentence |
| Alerts | **SweetAlert2 wrapped in `libs/sweetAlert.ts`** | Never call `Swal.fire` from a component |
| Uploads | `apollo-upload-client` + `browser-image-compression` | GraphQL multipart |
| Package manager | **yarn** (`yarn.lock` committed) | |
| Releases | `standard-version` → `CHANGELOG.md` | `yarn changelog` |

## A2. Formatting (non-negotiable)

`.prettierrc`:

```json
{ "tabWidth": 2, "useTabs": true, "singleQuote": true, "trailingComma": "all",
  "semi": true, "printWidth": 120, "endOfLine": "auto" }
```

- Tabs, single quotes, semicolons, trailing commas, 120 columns.
- **Relative imports only** — no `@/` aliases.
- No barrel `index.ts` files except `libs/auth/index.ts` and `scss/MaterialTheme/index.ts`.
- `// @ts-ignore` only at known-bad third-party boundaries, one line, localized.

## A3. Folder structure

```
apollo/
  client.ts              Apollo client factory, links, SSR-safe singleton
  store.ts               reactive vars (userVar, themeVar, unreadMessagesVar …)
  user/{query,mutation}.ts     public operations
  admin/{query,mutation}.ts    admin operations
libs/
  auth/index.ts          logIn / signUp / logOut / updateUserInfo
  config.ts              the ONLY place env vars are read; static option lists; Messages
  utils.ts               pure helpers + shared mutation handlers
  sweetAlert.ts          every alert / toast / confirm
  server.ts              server-side GraphQL fetch for getServerSideProps (SEO pages)
  socket.ts              websocket singleton (when the app has realtime)
  hooks/                 useDeviceDetect.ts …
  enums/                 <domain>.enum.ts — mirrors backend enums exactly
  types/<domain>/        <domain>.ts | <domain>.input.ts | <domain>.update.ts
  components/
    layout/              withLayoutBasic / withLayoutFull / withLayoutHome / withAdminLayout
    common/              cross-feature cards and widgets
    <feature>/           one folder per feature (product, member, mypage, community, cs …)
    admin/<section>/     admin-only components
    Top.tsx Footer.tsx   global chrome at the components root
pages/                   routes only — thin, no business logic
  _app.tsx _document.tsx index.tsx
  <entity>/{index,detail}.tsx
  account/join.tsx       combined login + signup
  _admin/**              underscore = not a public route family
scss/
  app.scss variables.scss reset.scss
  pc/main.scss           manifest importing every desktop partial
  pc/<page>/…            one partial per page / feature
  mobile/main.scss
  MaterialTheme/{index,typography,shadow}.ts
public/
  img/… locales/{en,kr,ru}/common.json
```

**`pages/` holds routes, `libs/` holds everything else, `apollo/` holds the data contract.** A page
never defines a reusable component.

## A4. Naming

| Kind | Convention | Example |
| --- | --- | --- |
| Component file | `PascalCase.tsx` | `ProductCard.tsx` |
| Page file | lowercase / `index.tsx` | `pages/product/detail.tsx` |
| Hook | `useCamelCase.ts` | `useDeviceDetect.ts` |
| Enum module | `kebab.enum.ts` | `board-article.enum.ts` |
| Type module | `<domain>.ts`, `.input.ts`, `.update.ts` | `product.input.ts` |
| Layout HOC | `withLayoutX` | `withLayoutBasic` |
| Handler | `<action>Handler` | `likeProductHandler`, `sortingHandler` |
| GraphQL document | `SCREAMING_SNAKE` const | `GET_PRODUCTS`, `LIKE_TARGET_PRODUCT` |
| Props interface | `<Component>Type` | `ProductCardType` |
| CSS class | kebab, semantic | `card-config`, `filter-config`, `no-data` |
| Page root | `id` | `id="product-list-page"`, `id="pc-wrap"` |

Entity fields are prefixed (`productTitle`, `memberNick`); list inquiries are
`<Entities>Inquiry { page, limit, sort?, direction?, search }`; list results are
`{ list, metaCounter: [{ total }] }`.

## A5. Types

```ts
// libs/types/product/product.ts — entity as returned by the API
export interface Product {
	_id: string;
	productStatus: ProductStatus;      // enums, never raw strings
	productTitle: string;
	productPrice: number;
	productDesc?: string;              // optional == nullable in the backend
	memberId: string;
	createdAt: Date;
	updatedAt: Date;
	/** from aggregation **/           // mark aggregation-only fields
	meLiked?: MeLiked[];
	memberData?: Member;
}
export interface Products { list: Product[]; metaCounter: TotalCounter[] }
```

- `<domain>.input.ts` holds create input + `<Entities>Inquiry` (nested search interface is private,
  named by initials: `PISearch`).
- `<domain>.update.ts` holds `<Entity>Update` with `_id` required, everything else optional.
- Enums mirror the backend one-to-one. `common.enum.ts` holds `Direction` and `Message`.
- Do **not** select GraphQL fields the backend declares non-null but can't always resolve — check
  the resolver before adding a field to a selection.

## A6. Apollo / data layer

- Link chain: `from([errorLink, tokenRefreshLink, splitLink])`; `splitLink` sends subscriptions to
  the WS link and everything else to `authLink.concat(uploadLink)`.
- Links are built **only in the browser** (`typeof window !== 'undefined'`) so SSR stays safe.
- Documents are split by audience (`apollo/user`, `apollo/admin`) and by kind (`query.ts`,
  `mutation.ts`), grouped with banner comments, operation names in PascalCase, **one `$input`
  variable** per operation, selections written out explicitly.
- Admin screens import only from `apollo/admin/*`.

```tsx
/** APOLLO REQUESTS **/
const [likeTargetProduct] = useMutation(LIKE_TARGET_PRODUCT);
const { loading, refetch } = useQuery(GET_PRODUCTS, {
	fetchPolicy: 'cache-and-network',
	variables: { input: searchFilter },
	notifyOnNetworkStatusChange: true,
	onCompleted: (data: T) => {
		setProducts(data?.getProducts?.list);
		setTotal(data?.getProducts?.metaCounter[0]?.total ?? 0);
	},
});
```

- Global state is a new `makeVar`, read with `useReactiveVar`. Never add Redux or a Context for it.
- Auth-critical mutations in `libs/auth` use `apolloClient.mutate({ fetchPolicy: 'network-only' })`.

## A7. Auth flow

1. `logIn` → `LOGIN` mutation → token.
2. `updateStorage` → `localStorage.accessToken` + a `login` timestamp (cross-tab signal).
3. `updateUserInfo` → decode → fill `userVar` with every field defaulted.
4. `logOut` → remove token, set `logout` timestamp, reset `userVar`.
5. Every layout HOC rehydrates `userVar` from the stored token on mount.
6. Pages guard by redirect in `useEffect`; `withAdminLayout` renders `null` unless the user is ADMIN
   and waits for the auth-loaded flag before redirecting.
7. Social login (Google / Kakao / Telegram) goes through **one** `socialLogin` mutation; the
   frontend asks `getAuthProviders` which buttons to show, so an unconfigured provider is hidden.

The client guard is a convenience. **The API guard is the rule.**

## A8. Page anatomy (copy this skeleton)

```tsx
export const getStaticProps = async ({ locale }: any) => ({
	props: { ...(await serverSideTranslations(locale, ['common'])) },
});

const ProductList: NextPage = ({ initialInput = DEFAULT_INPUT }: ProductListProps) => {
	const device = useDeviceDetect();
	const router = useRouter();
	const [searchFilter, setSearchFilter] = useState<ProductsInquiry>(
		router?.query?.input ? JSON.parse(router.query.input as string) : initialInput,
	);

	/** APOLLO REQUESTS **/

	/** LIFECYCLES **/

	/** HANDLERS **/

	if (device === 'mobile') return <div id="product-list-page">{/* mobile */}</div>;
	return <div id="product-list-page">{/* desktop */}</div>;
};

export default withLayoutBasic(ProductList);
```

Rules:

- The three section comments `/** APOLLO REQUESTS **/`, `/** LIFECYCLES **/`, `/** HANDLERS **/`
  appear in that order and spelling.
- Every page exports `getStaticProps` with `serverSideTranslations`; public SEO pages use
  `getServerSideProps` through `libs/server.ts` instead.
- List/filter/pagination state is serialised into the URL (`?input=<json>`) and re-read on router
  change. **The URL is the source of truth for list state.**
- Sub-page tabs use `router.query.category` with a default.
- Internal links use `next/link` with an object href.
- Empty state is the standard `.no-data` block.
- Banner title/description per route is resolved in **one** `useMemo` switch on `router.pathname`
  inside the layout.

## A9. Handlers and errors

```ts
const likeProductHandler = async (id: string) => {
	try {
		if (!user._id) throw new Error(Message.NOT_AUTHENTICATED);
		await likeTargetProduct({ variables: { input: id } });
		await refetch({ input: searchFilter });
		await sweetTopSmallSuccessAlert('success', 800);
	} catch (err: any) {
		console.log('ERROR, likeProductHandler:', err.message);
		sweetMixinErrorAlert(err.message).then();
	}
};
```

- Every handler: guard → mutate → refetch → toast, with `console.log('ERROR, <handlerName>:', …)`.
- All user feedback goes through `libs/sweetAlert.ts`.
- Backend error strings are mapped to friendly messages in one `switch` in `libs/auth`.

## A10. Styling

- `_app.tsx` imports exactly `scss/app.scss`, `scss/pc/main.scss`, `scss/mobile/main.scss`.
- **Adding a page = one SCSS partial + one `@import` line** in the manifest.
- Everything is scoped under `#pc-wrap` / `#mobile-wrap`, then the page id, nested like the JSX.
- MUI is themed centrally in `scss/MaterialTheme/index.ts` (palette + `styleOverrides` that flatten
  MUI defaults). One-off tweaks use `sx`; anything repeated goes to SCSS.
- Responsive SCSS is preferred for new projects over separate mobile trees (see A12).

## A11. i18n

- `next-i18next`, `defaultLocale: 'en'`, `locales: ['en','kr','ru']`, `localeDetection: false`.
- Keys are the English sentence. **Adding copy = adding the key to all three files.**
- Locale code `kr` makes `Intl.PluralRules` fall back to en-US — give plural keys in `kr` every
  suffix (`_one/_few/_many/_other`). HTML `lang` is `ko`.
- Translated *data* from the API is `{ en, ru, kr }`; the server fills blanks from `en`.
- Language switch uses inline SVG flags (Windows cannot render flag emoji).

## A12. Admin area

- Routes under `pages/_admin/**`; components under `libs/components/admin/<section>/`.
- `withAdminLayout` (AppBar + Drawer + menu list) gates by role.
- Admin tables: MUI `Table` + `headCells` array + `descendingComparator` recipe.
- If the admin needs to be "its own website", it becomes a separate app or route group with its
  own login — decide before building, don't grow it inside the public layout.

## A13. Do **not** carry these over from the reference code

- `any` on page props and handler params — type them.
- `defaultProps` on function components — use default parameters.
- `console.warn` on every Apollo operation — remove or gate behind a debug flag.
- A stubbed token-refresh link that never refreshes — implement it or remove it.
- Raw `<img>` without size — use `next/image` or set width/height.
- User-agent device detection after mount (first paint is always desktop) — prefer responsive CSS.
- Empty `useEffect` and empty `/** APOLLO REQUESTS **/` placeholders.

---

# Part B — App Router dashboards

Owner dashboards in front of an existing API (Python FastAPI or Nest).

- Next.js 16 App Router, TypeScript, **plain CSS with design tokens** (light default + dark toggle
  stored in a cookie), one font (e.g. Manrope).
- The dashboard is **read-only by default.** Write actions are added only when asked, and they call
  the same backend function the other channel (e.g. Telegram buttons) uses, so the two can never
  disagree.
- Login: one `DASHBOARD_PASSWORD` + a signed, httpOnly session cookie. The API is called
  server-side with a bearer token; the token never reaches the browser.
- Bind to `127.0.0.1` and put it behind the reverse proxy / tunnel.
- `proxy.ts` (middleware) buffers request bodies; exclude upload routes from it and stream uploads
  through a route handler. Serve video through a route handler that supports `Range`.
- Read the shared root `.env` with `@next/env` when the dashboard lives inside a Python repo.
- Server Components fetch; Client Components only where there is interaction.
