# Svelte 5 Standards

Used for apps that run **on a Raspberry Pi inside a restaurant**, where bundle size and runtime
cost matter. The frontend is built on a PC and its `dist/` is served by the FastAPI backend.
Reference repos: `navai-front`, `silkroad-new/frontend`.

---

## 1. Stack

- Svelte 5 **runes** + TypeScript + Vite, **multi-page app** (one `index.html` per page:
  `menu/`, `admin/`, `printers/`).
- `npm run build` = `svelte-check --fail-on-warnings` + `vite build`. Warnings fail the build.
- No component library. Icons are inlined lucide paths in one `Icon` component.
- Built `dist/` is copied to the device; **after any front change, rebuild and redeploy `dist`**.

## 2. Layout

```
menu/ admin/ printers/        index.html per page
src/lib/api/client.ts         the only fetch wrapper; ApiError(status, code, detail)
src/lib/api/types.ts          hand-written mirrors of the backend JSON (camelCase)
src/lib/api/public.ts         customer + waiter endpoints
src/lib/api/admin.ts          admin endpoints (send the stored token)
src/lib/config.svelte.ts      /api/config once per page + brand colour
src/lib/format.ts             money, dates, localized(), thumb()
src/lib/i18n/menu.ts          customer strings (en / ru / uz / ko)
src/components/               Modal, Toast, ConfirmButton, Icon, Skeleton, Flag
src/apps/<page>/              App.svelte, state.svelte.ts, page components
```

## 3. Conventions

- **Presentation only.** Totals, prices, stock, report numbers come from the API. Client checks are
  hints on top of server rules.
- **All HTTP through `request()`**; add typed functions to `public.ts` / `admin.ts` and types to
  `types.ts` *in the same change as the backend*.
- **Errors:** catch `ApiError`, switch on its `code` (the backend's `detail.error` snake_code) and
  show a translated message on customer pages, `errorText(err)` on staff pages.
- **Staff UI copy is English. Customer UI is translated** — every new key goes into all four languages.
- **Money** always through `money(n, appConfig.currencySuffix)`. Never hard-code a currency.
- **No emoji in the UI.** Use text or inline SVG (flags are SVG components).
- **Brand colour** is one `--accent` token (+ derived strong / soft shades). Cache-bust brand assets
  (`/logo.svg?v=<hash>`) because the CDN caches them for hours.
- State lives in `state.svelte.ts` modules using runes; components stay thin.

## 4. UX rules we agreed on

- **Undo, not "Are you sure?".** Routine admin actions run immediately and offer Undo in the toast.
  Confirmation dialogs only for destructive, non-reversible actions.
- **Pages, not sheets.** Cart, Account, My Orders are tab pages. A page that holds typed input
  (cart delivery details) stays mounted while hidden so input survives a tab switch.
- **Skeletons until first data** (`hallLoaded` / `menuLoaded` flags), never the empty-state text.
  A reload keeps old data, dimmed, and ignores stale answers by request id.
- Tabs open on the URL hash at load (no flash of the first tab) and fade in with **opacity only** —
  a transform re-anchors `position: fixed` dialogs.
- Dropdowns inside scroll areas are **portaled to `<body>`** with fixed positioning so they aren't clipped.
- Charts with no data render a fixed-height blank box — an empty SVG with a tiny viewBox stretched
  to 100% width renders giant axis labels.
- Readable greys only (`--muted`, `--faint` pass WCAG AA on white); staff text ≥ 12px.
- Remove features the client said they don't want (search bar, favourites, cashback). Don't re-add
  them "for completeness".

## 5. Gotchas

- Admin token in `localStorage` (`<app>_admin_token`); a 401 signs out.
- Staff ordering embeds the customer page in an iframe (`?embed=1`); language sync uses
  `postMessage` restricted to the same origin.
- Third-party scripts (Daum postcode, Google GSI, Telegram widget) must have a fallback when they
  fail to load (plain text field, hidden button).
- Inside a Telegram Mini App (`tgWebAppData` in the hash) sign the customer in silently.
