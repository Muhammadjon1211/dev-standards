# ESC/POS Thermal Printer Standards

Kitchen and hall receipt printers attached to the restaurant Pi over USB. Every order prints a
kitchen ticket (food only, with a beep) and a hall ticket (everything, with the total). Reference
code: `navai-back/app/printing/`, `app/services/print_queue.py`.

---

## 1. Connecting a printer

Printers show up in one of two ways — find out which before writing any config:

| Kind | Device | Example |
| --- | --- | --- |
| USB printer class | `/dev/usb/lp0` | Bixolon SRP-330II |
| USB CDC serial ("virtual COM port") | `/dev/ttyACM*` | Howbest / AHAPOS receipt printers (lsusb may mislabel them) |

**Always map printers to stable names with a udev rule, by USB serial number**, so kitchen and hall
never swap after a reboot or replug:

```
# /etc/udev/rules.d/97-<app>-printers.rules
SUBSYSTEM=="tty", ATTRS{serial}=="<kitchen-serial>", SYMLINK+="escpos-kitchen", GROUP="lp", MODE="0660", RUN+="/bin/stty -F /dev/%k raw"
SUBSYSTEM=="tty", ATTRS{serial}=="<hall-serial>",    SYMLINK+="escpos-hall",    GROUP="lp", MODE="0660", RUN+="/bin/stty -F /dev/%k raw"
```

- **Serial printers need `stty raw`** — otherwise the tty turns LF into CRLF and corrupts ESC/POS binary.
- The service user is in group `lp`.
- `.env` points at the symlinks: `KITCHEN_PRINTER_DEVICE=/dev/escpos-kitchen`,
  `HALL_PRINTER_DEVICE=/dev/escpos-hall`.
- Label the physical printers to match ("KITCHEN", "HALL").

## 2. The print queue (never lose a ticket)

Every ticket is **first a row in `print_jobs`**, then printed. Nothing is printed straight from the
request.

- One worker per printer, oldest job first; the kitchen never blocks the hall.
- Before each job, ask the printer its status with **DLE EOT** (`0x10 0x04 n`): unplugged / no paper /
  cover open. No answer = assume ready.
- Not ready → wait and retry every `PRINT_RETRY_S` (5 s). Tickets come out by themselves after a
  paper refill or reconnect.
- A job that raises an exception (bad data) is marked `failed` at once — don't retry forever.
- Jobs older than `PRINT_JOB_MAX_AGE_MIN` (60) expire; an hour-old ticket is noise in a kitchen.
- An order becomes `printed` only after the printer (or agent) confirms.
- The admin top bar shows each printer's problem and the waiting count
  ("Hall printer: no paper · 2 waiting").
- **Printing is blocking I/O** — run it in an executor, never on the event loop.
- Auto-print is triggered from the DB `LISTEN/NOTIFY` callback, not inside the HTTP request.

### Modes

| `PRINT_MODE` | Where the app runs | How tickets reach the printer |
| --- | --- | --- |
| `local` | On the Pi | Workers write to the USB devices directly |
| `agent` | On a VPS | Jobs pushed over SSE to a print agent on the Pi (`x-print-token`); the agent prints in order, reports each result, retries a failed print for a few minutes, and **never prints a resent job twice** |

`PRINTER_ENABLED=0` makes printing succeed without hardware — the default for development.

## 3. Ticket content

- **Kitchen:** order number big, order type (DINE-IN table / TAKEOUT pickup time / DELIVERY), food
  items only, large text, beep. Category flag `kitchen_printer` and product flag `kitchen_only`
  decide routing.
- **Hall:** everything, prices, total, delivery details for delivery orders, silent.
- Order # and dish names flush **left**; date/time, qty, price, TOTAL flush **right**, done with
  padding to the column count — not with ESC a (alignment) commands.
- Times print in `RESTAURANT_TZ`, whatever the server clock says.
- The printer code page has **no Hangul / Cyrillic guarantee** — print English names
  (`address_en`, base English menu columns). Translated names are for screens.
- Never change the bytes of a ticket the client already approved without a fresh on-paper check.

## 4. Layout widths — measure, don't guess

- Column counts are settings: `HALL_LINE_COLS`, `KITCHEN_LINE_COLS` (typical 80 mm paper: 42–48
  normal columns; double-height text with extra character spacing fits fewer).
- Measure on the actual printer: print a ruler line, count what fits, set the value, print again.
- Dividers are as wide as the content lines.
- The left non-printable strip is physical; `GS L` won't remove it.
- **Ask for a photo of the printed ticket** before and after any layout change.

## 5. Buzzer and extras

- Many printers ignore `BEL`. Xprinter-style models answer `ESC B n t` (n beeps, t × 50 ms) — test
  by ear.
- Printers can't play melodies; use the admin screen or a speaker on the Pi for richer alerts.
- QR slips for tables print on the hall printer in image mode.

## 6. Debugging checklist

1. `ls -l /dev/escpos-*` — symlink present? (missing → unplugged or udev rule wrong)
2. `dmesg | tail` — "USB disconnect" means cable/power, not code.
3. Is the service user in `lp`? Is `DeviceAllow=` set in the unit (remove it)?
4. `stty -F /dev/escpos-hall -a` — is it raw?
5. Print a test ticket with a fake order from a script, per printer.
6. Check `print_jobs` for `failed` / waiting rows and the admin status endpoint.
