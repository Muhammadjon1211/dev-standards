# UI / UX Standards (all frontends)

Design and copy rules that hold across Next.js, React and Svelte projects.

---

## 1. Copy

- **Minimal copy.** No subtitles or descriptive paragraphs under headings unless asked. A heading
  and the content is enough.
- Messages are written for the end user (student, customer, restaurant staff), not for developers.
- Staff/admin UI may be single-language (English); customer UI is translated (en / ru / uz / ko / kr
  depending on the market).
- Never show a raw enum (`IN_REVIEW`) — always a label (`In review`).

## 2. Every screen has four states

Loading (skeleton shaped like the content) → Error (inline, `role="alert"`) → Empty (with a next
action) → Content. Never flash the empty state while loading.

## 3. Feedback

- Inline errors for forms; toasts for background actions.
- **Undo instead of "Are you sure?"** for routine, reversible actions.
- Modal confirmation only for destructive, irreversible actions.
- No native `alert` / `confirm` / `prompt`.
- Buttons show progress (`Saving…`) and are disabled while a request is in flight.

## 4. Visual system

- Colours are **tokens**, defined once, with light and dark palettes. No hex in component rules.
- One accent / brand colour per product, configurable (`BRAND_COLOR` / `--accent`) so a white-label
  copy can rebrand without code changes.
- One display font + one text font at most. Load from Google Fonts or self-host.
- Cards: consistent radius, consistent spacing scale. Prefer whitespace to borders.
- No emoji as UI icons. Inline SVG (lucide) or text. Flags are SVG — Windows does not render flag emoji.
- Prices and key numbers stay high-contrast (dark), not muted.

## 5. Layout and interaction

- Mobile first for customer apps (QR menus, Telegram Mini Apps): bottom tabs, full-width primary
  buttons, thumb-reachable actions.
- Keep the **full-width "+ Add" button** with a − n + stepper on dish/product cards; small corner
  "+" icons were rejected.
- Use pages instead of stacked popups/sheets for major flows (account, cart, orders). Small info
  popovers (ⓘ) are fine.
- Respect `prefers-reduced-motion`. Animate opacity rather than layout.
- Visible focus states everywhere; keyboard-operable dropdowns and dialogs.

## 6. Accessibility checklist

- Colour contrast ≥ WCAG AA for text.
- `aria-label` on icon-only buttons; `aria-hidden` on decorative icons.
- Landmarks: skip link, `<main>`.
- Dialog: focus trap, Esc, restore focus.
- Live region for toasts.

## 7. Working from references

- The client's screenshots / reference folder define which screens matter. Match them before
  adding anything of our own.
- When judging something physical (a printed receipt, a device screen), **ask for a photo early**
  instead of guessing from code.
- Apply the UX laws (Hick, Fitts, Jakob, proximity, Miller, Doherty) when reviewing a screen —
  fewer choices, bigger targets, familiar patterns, fast feedback.
