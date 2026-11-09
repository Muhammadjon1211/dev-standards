# LLM Integration Standards

How we put a language model inside a product feature: menu translation, AI summaries, chat
assistants, email drafting. Reference implementations: Chayhana / Navai menu translator,
Waeyo HQ AI chat, Sales-Agent drafter, Modu recommendations and try-on.

---

## 1. Decide if it needs a model at all

- **If deterministic code can do it, write code.** Prices, totals, discounts, eligibility,
  recommendations from behaviour data, coaching copy from templates — all code.
  Modu recommendations are content-based + collaborative filtering + popularity, no paid AI.
- Use a model for language: translating, summarising, drafting, classifying free text, answering
  questions about a known document.
- **A model never produces a number the customer pays.** The LLM writes the words; Python/TS
  computes the price and inserts it.

## 2. Where it runs

- **Always server-side.** API keys never reach the browser, never go into `NEXT_PUBLIC_*` / `VITE_*`.
- One module owns the client (`services/translator.py`, `pages/api/hq-chat.ts`).
- Model name, effort and limits come from env (`TRANSLATE_MODEL`, `AI_MODEL`, `TRANSLATE_PER_RUN`)
  with a sensible default in config. **Never hardcode a model id in business code.**
- Default provider: Anthropic Claude. When a client wants free models, the same route also speaks
  the **OpenAI-compatible** API (`AI_BASE_URL`, `AI_API_KEY`, `AI_MODEL`) — Gemini, Groq,
  OpenRouter, Ollama. `AI_MODEL` may be a **fallback list**, because free models are often busy or retired.
- Pick the cheapest model that does the job: small/fast model (or low effort) for translation and
  classification; larger model for drafting and reasoning.

## 3. Degrade, don't crash

- No API key → the feature is **off**, not broken: rows stay "Translating…", the chat answers from an
  offline fallback, the button is hidden.
- Network / 429 / 5xx → keep the work pending and retry on the next run.
- Bad key or unknown model → pause the job (e.g. 15 min) and log loudly.
- A refusal → clear the pending flag and keep the original text.

## 4. Background jobs over request-time calls

Translation pattern (copy it):

1. A write that changes translatable text (name, description, ingredients, variant labels — not
   price or photo) sets `translate_pending = true`.
2. A background task every 30 s takes up to `TRANSLATE_PER_RUN` pending rows.
3. It asks the model for **structured JSON output** (JSON schema), all target languages at once.
4. It writes the result **only if the source text is unchanged since it was read** (optimistic check).
5. The UI shows "Translating…" and polls until done.

The admin writes once, in any language. Base columns become English (what tickets print and what
code matches on); each language keeps its own column.

## 5. Prompts and rules

- Prompts live in code or in a versioned file next to it, not in the database.
- **Owner-editable rules files** (`RULES.md`, `aichatbot.md`) are read on every request, so the
  owner changes behaviour without a deploy. Separate clearly:
  - *facts the model may state* (payment terms, lead times) — written in a strict format;
  - *rules for the agent* (when not to answer automatically) — never copied into output.
- A figure not written in the allowed format authorises nothing: the model must defer, not invent.
- Ground answers in provided data (store snapshot, menu, price list). Tell the model to say it
  doesn't know rather than guess.
- Output language follows the user; unsupported languages get English plus a flag for the owner.

## 6. Privacy, cost and limits

- Personal photos and documents sent to a model are **not stored** — return the result (e.g. as a
  data URL) and discard the input.
- Rate-limit per user (e.g. try-on 10/hour, members only).
- Log usage, not content.
- Check the **licence** of any model or hosted Space you call. Non-commercial licences (e.g.
  CC BY-NC-SA) are acceptable for demos only, never for a paying client.

## 7. RAG and vision projects

- RAG services: FastAPI + Pydantic + a vector store (Qdrant); chunking, embedding and retrieval in
  separate modules; the generator is swappable; every answer cites its chunks.
- Vision (YOLO detection, image-to-mesh): pipelines are scripts with a `run_pipeline.py` entry,
  notebooks for exploration only, `requirements.txt` pinned, tests for the deterministic steps.

## 8. Checklist before shipping an AI feature

- [ ] Works with the key unset (feature hidden / off).
- [ ] Model id and limits configurable by env.
- [ ] Structured output validated before it is written.
- [ ] Money and business rules computed in code.
- [ ] Retries and pause-on-bad-key handled.
- [ ] No personal input stored.
- [ ] Licence of model / service checked.
