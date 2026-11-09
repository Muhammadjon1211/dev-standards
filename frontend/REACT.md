# React (Vite SPA) Standards

For single-page apps with no server of their own: React 19 + TypeScript + Vite, talking to a
NestJS REST API or an Express API. Reference repo: `kora/frontend`. Lessons also come from the
original FoodBot (Vite + React Telegram Mini App).

---

## 1. Stack

| Concern | Choice |
| --- | --- |
| Build | Vite 7 (`npm run build` = `tsc -b && vite build`) |
| Language | TypeScript strict |
| Router | `react-router-dom` v7, browser history |
| Icons | `lucide-react` (or inlined lucide paths) |
| Styling | **One global plain-CSS file, token-driven, light + dark.** No Tailwind, no CSS modules, no CSS-in-JS |
| State | Local `useState` + a few contexts (auth, theme, toasts). No Redux / React Query / Zustand |
| Node | Pinned in `.nvmrc` / `.node-version`, enforced by `scripts/check-node.cjs` in `predev`/`prebuild` |
| Package manager | One per repo (npm → `package-lock.json`). Never create a second lockfile |

## 2. Layout

```
index.html
src/main.tsx              StrictMode > ThemeProvider > BrowserRouter > ToastProvider > AuthProvider > App
src/App.tsx               every route + the Protected wrapper
src/api/client.ts         the ONLY fetch wrapper; ApiError
src/api/types.ts          hand-written mirrors of backend types
src/auth/AuthContext.tsx  session state, login / register / logout
src/lib/format.ts         enum label maps, money and date formatters
src/content/              editorial copy, kept out of components
src/components/           Layout, Modal, Toast, Skeleton, StatusBadge, Avatar …
src/pages/                one page per file, named export
src/styles.css            every style in the app, numbered sections
```

Pages are named exports (`export function ProgramsPage()`), imported explicitly in `App.tsx`.
No barrels.

## 3. API access

**All network calls go through `api<T>(path, options)` in `src/api/client.ts`.** Never call `fetch`
from a page or component. The wrapper:

- prefixes `import.meta.env.VITE_API_URL`;
- attaches `Authorization: Bearer <token>` from storage;
- sets `Content-Type: application/json` **unless the body is `FormData`** (uploads must not set it);
- flattens the backend error (`message` may be a string *or* an array) into `ApiError { message, status }`;
- clears the token on 401;
- returns `undefined` for 204;
- **aborts after 8–10 s** with an `AbortController` and reports "Request timed out" separately from
  other errors — a stale tunnel URL otherwise hangs forever with no error.

Backend messages are written for users: show `ApiError.message`, don't replace it with "Error".

`src/api/types.ts` is **hand-maintained**. When the backend changes a model, enum or response,
update this file in the same change, or `tsc` will pass on stale assumptions.

## 4. Page conventions — the four states

```tsx
const [data, setData] = useState<Program[] | null>(null);
const [loading, setLoading] = useState(true);
const [error, setError] = useState('');

async function load() {
	setLoading(true);
	try { setData(await api<Program[]>('/catalog/programs')); setError(''); }
	catch (e) { setError((e as ApiError).message); }
	finally { setLoading(false); }
}
useEffect(() => { void load(); }, [deps]);
```

Every data page renders **all four**:

1. **Loading** — a skeleton shaped like the real content. Never render the empty state while a
   request is in flight (it flashes "nothing here").
2. **Error** — `.form-error` with `role="alert"` carrying the API message.
3. **Empty** — `.empty-state` with a next action.
4. **Content.**

Rules that came from real bugs:

- Every `.then()` has a `.catch()`. An unhandled rejection is a blank page.
- **Validate and early-return *before* `setSaving(true)`** — otherwise the spinner never clears.
- Disable submit buttons while in flight and label them (`Saving…`, `Uploading…`).
- After a save, **navigate first, then refetch without awaiting** — awaiting the refetch makes the
  save feel slow.
- `import { type ReactNode } from 'react'` — never `React.ReactNode` without importing React
  (it is `undefined` at runtime → blank page, no error).
- React 19 `StrictMode` runs effects twice in dev. That is expected; do not add a ref guard.
- Reset `event.target.value` after a file input fires, or picking the same file twice does nothing.
- Wrap every `localStorage` / `sessionStorage` access in `try/catch` (private mode throws).

## 5. Where errors and confirmations go

- Form errors render inline next to the fields (`role="alert"`).
- Background actions (save, upload, status change) report through `useToast().push(kind, title, detail?)`.
- Page-level load failures render at the top of the page.
- **Never** `window.alert`, `window.confirm`, `window.prompt`. Use `Modal`.
- **No silent `catch {}`.** A failed admin action shows a visible error state.

## 6. Auth and storage

- `useAuth()` returns `{ user, loading, login, register, logout }` and throws without its provider.
  On mount it calls `/auth/me` if a token exists and drops the token if that fails.
- Customer tokens: `localStorage`.
- **Admin / staff tokens: `sessionStorage`** when several accounts are used on one machine —
  `localStorage` is shared by all tabs, so a new `/admin` tab would log in as the previous
  restaurant.
- Role is never chosen by the client. The UI only hides what the API would refuse.

## 7. Never render a raw enum

The API returns `SCREAMING_SNAKE_CASE`. Every user-facing value goes through `src/lib/format.ts`
(`statusLabels`, `label(value)`, `money`, `shortDate`, `relativeDate`). Adding a status touches
three places: `api/types.ts`, the label map, and a `.status-<lowercase>` colour rule.

## 8. Styling

`styles.css` is one global stylesheet in numbered sections (tokens, base, topbar, buttons/forms,
badges, panels, one section per page, toasts, modal, animations, responsive).

**Everything is driven by semantic CSS custom properties** — `--bg`, `--surface`, `--border`,
`--text`, `--text-muted`, `--brand`, `--accent`, status pairs `--ok-bg/--ok-fg`, `--warn-*`,
`--danger-*`, radii `--r-*`, `--shadow-*`, `--ease`.

**Never hardcode a colour in a component rule.** Dark mode = redefining tokens in exactly three places:

```css
:root { /* light palette */ }
@media (prefers-color-scheme: dark) { :root:not([data-theme="light"]) { /* dark */ } }
:root[data-theme="dark"] { /* dark, explicit toggle */ }
```

- A colour defined only inside a media query or a `[data-theme]` block is a bug.
- A rule like `:root[data-theme="dark"] .card { … }` fires only for the explicit toggle and silently
  does nothing for *system* dark — give the value a token instead.
- Semantic class names (`.program-card`, `.panel`, `.button.ghost`), no inline `style` except truly
  dynamic values, no utility-class soup. Reuse existing primitives before inventing new ones.
- Breakpoints live in one responsive section (e.g. 1180 / 1000 / 700 px). End it with a
  `prefers-reduced-motion` block.
- Theme choice (`light | dark | system`) persists in storage.

## 9. Accessibility (required, not optional)

- Skip link + `<main id="main">` landmark in the shell.
- Icon-only buttons have `aria-label`; decorative icons have `aria-hidden="true"`.
- Dropdowns: `aria-expanded`, `aria-haspopup`, click-outside and Esc.
- Dialogs go through `Modal`: `role="dialog"`, `aria-modal`, focus trap, Esc, scroll lock, focus restore.
- Toasts in an `aria-live="polite"` region; errors `role="alert"`.
- Progress bars: `role="progressbar"` + `aria-valuenow/min/max`.
- `:focus-visible` is styled globally — never remove outlines.

## 10. Environment and dev server

- Only `VITE_*` variables reach the client, and everything there is public. No secrets in `.env`.
- `VITE_*` values are **baked in at build/start**. Changing `.env` needs a dev-server restart and a
  rebuild for production.
- When the app is opened from a phone (Telegram Mini App, QR), `VITE_API_URL` must be the public
  URL, never `localhost` — `localhost` resolves on the customer's phone.
- Behind a Cloudflare tunnel, set `server.allowedHosts: true` in `vite.config.ts`.
- If the backend allows a single CORS origin, pin `server.host` (`localhost` ≠ `127.0.0.1`).
- `vite.config.js` / `.d.ts` emitted by `tsc` shadow `vite.config.ts` — gitignore and delete them.

## 11. Assets and licensing

- Images that ship with the app are committed under `public/`, never hotlinked.
- Use freely licensed sources only and credit them in a `CREDITS.md` next to the files.
- Every remote image has a generated fallback (initials avatar, SVG crest) and keeps a broken state,
  so a 404 falls back instead of rendering a broken image.

## 12. Telegram Mini Apps

- Submit orders with `fetch()` to the backend, **not** `tg.sendData()` — it closes the Mini App
  before the success screen renders.
- Outside Telegram, the user id is `undefined` → falls back to `0`. Backend checks must use
  `=== undefined || === null`, not `!id`.
