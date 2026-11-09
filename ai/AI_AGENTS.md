# AI Agent Standards

Standards for **business agents** that act for a client — answering enquiries, drafting quotes,
editing and publishing social posts, replying to comments and DMs. Reference projects:
`sales-agent` (Gmail / WhatsApp quotes, Telegram approval) and `smm-agent` (Instagram / Telegram /
voice, video editing, Telegram approval).

---

## 1. Human in the loop by default

- The agent **drafts**; the owner **approves**. Approval happens on the owner's phone through
  Telegram inline buttons (Approve / Reject / Edit with a note).
- Anything that leaves the business — an email, a quote, a published post, a reply to a customer —
  needs approval unless the owner has explicitly switched that action to automatic.
- **Start in dry run.** A fresh install never sends anything. Going live is an explicit setting.
- Escalate instead of guessing: unknown size, new customer over a threshold, a request for a
  sample, a language we don't support → a flagged card for the owner, no automatic reply.

## 2. Code decides, the model writes

- Prices, discounts, totals, currency and container math are computed in Python from the owner's
  price list (Excel workbook). The model receives the numbers and writes the sentence.
- Facts the agent may state (payment terms, lead time, quote validity, Incoterms) come only from
  the owner's rules file. A figure not there is deferred, never invented.
- Business rules for an action live in **one module** (`post_stage` / `post_actions`). Telegram
  buttons and the dashboard both call it, so the two channels can never disagree.

## 3. Rules file the owner edits

- `RULES.md` (or similar) in plain language, read on every draft.
- Three sections, clearly separated:
  1. **Notes for the owner** — what is configured elsewhere (prices live in Excel). No code reads it.
  2. **Things I always tell customers** — the only facts the writer may state, in a strict format
     ("15 working days", "50%").
  3. **When I will not answer automatically** / **Never do** — rules for the agent, never quoted.
- A `remember` command in the owner chat appends facts in the same format.

## 4. Channels

- Each channel (Gmail, WhatsApp Cloud API, Instagram Graph API, Telegram, Telegram Business,
  Twilio voice) is an adapter with the same interface: receive → normalise → thread → hand to the agent.
- Webhooks arrive through a Cloudflare tunnel; verify every webhook signature.
- Store a `threads` table: customer name, channel, history, status.
- Media (video, audio) is processed with ffmpeg / Whisper on the server; uploads are streamed, not buffered.

## 5. Dashboard

- Next.js App Router + plain CSS, bound to `127.0.0.1`, password login + signed cookie.
- Talks to a **read-only** API (FastAPI, bearer token) by default.
- Write actions only when the owner asks, and only through the shared action module.
- Shows: inbox, drafts with search, escalations, history, brand/config (read-only).
- Demo deployments use demo data, never a real client's inbox.

## 6. Ops

- Docker Compose with profiles (`dashboard`); compose project name, UID and ports from `.env`;
  start services by name.
- Secrets in `.env`; `.env.example` documents each one.
- Tests: `pytest` for the agent, node tests for the dashboard; CI on Linux. On Windows use
  `PYTHONUTF8=1`.
- Log every model call's purpose, token usage and outcome — not the customer's content.

## 7. Checklist for a new agent

- [ ] Dry run is the default.
- [ ] Every outbound action goes through approval (or an explicit, owner-set exception).
- [ ] Numbers computed in code; facts only from the rules file.
- [ ] Escalation rules exist and are tested.
- [ ] One action module shared by all channels.
- [ ] Webhook signatures verified.
- [ ] Dashboard read-only unless agreed otherwise.
