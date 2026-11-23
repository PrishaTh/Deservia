# Deservia

**Lifestyle-based rental matching for tenants and landlords** — instead of browsing hundreds of ads, both sides answer a short questionnaire and Deservia scores how well they would actually live together: budget, area, cleanliness, sleep schedule, guests, pets, smoking, and more.

Built for the **Brampton AI Hackathon** (Challenge 9). The app ships with an AI assistant that answers questions about a user's own listings and likes using live account data.

---

## Features

**Tenants**
- Questionnaire-driven profile (budget, area, move-in date, lifestyle habits).
- Swipe deck of scored listings with transparent reasons and "watchouts" for every match.
- Swiping saves instantly — likes land in **Liked listings**, passes are recorded separately, and changing a decision in the Review screen updates the saved choice.
- Side-by-side **compare** view of liked homes, with a score breakdown per listing.
- Requests to landlords, viewing scheduling (availability, propose, confirm, complete), rental application PDF, trust ratings and reporting.

**Landlords**
- Create and manage listings (rent, rooms, amenities, policies, availability, pause/lease states).
- See incoming tenant requests with the tenant's match score and reasons, then accept, decline, or move to screening.
- Viewing calendar, e-sign flow, and ratings after a completed tenancy.

**Admins**
- Moderation dashboard: reports, user suspension, listing state changes, data requests, audit log, match and fairness views.

**AI assistant** (`/api/chatbot`)
- Floating chat widget mounted in the app layout; tenant- and landlord-aware.
- Tool-calling over live data: `get_liked_listings`, `get_listing_details`, `get_my_listings`, `get_platform_faq`.
- Read-only by design: identity comes from the session cookie, tools only read, and no tool can create, edit, accept, or schedule anything.
- Provider-agnostic: Google Gemini by default (`gemini-3.1-flash-lite`, with `gemini-3.5-flash` / `gemini-3.8-flash` fallbacks), Anthropic Claude when `ANTHROPIC_API_KEY` is present.
- Guardrails: 30 messages per user per hour, 2 000-character messages, 4 tool steps, 18–25 s timeouts, and a friendly fallback message instead of an error dump.

---

## Tech stack

| Layer | Choice |
| --- | --- |
| Runtime | Cloudflare Workers (`compatibility_date` 2026-11-21) |
| API | Hono 4, TypeScript |
| Database | Cloudflare D1 (SQLite) via the `DB` binding, prepared statements, no ORM |
| Front end | React 19, React Router 7, Tailwind CSS 4 |
| Build | Vite 8, `@vitejs/plugin-react`, TypeScript 7 (separate app + worker projects) |
| AI | Google Gemini REST (function calling), optional Anthropic SDK |
| Tooling | Wrangler 4, `concurrently` |

**How it fits together:** the browser loads the React SPA from the Worker's `ASSETS` binding and calls relative `/api/*` paths with `credentials: "include"`, so the HTTP-only session cookie authenticates every request. `run_worker_first` sends those paths into the Hono app, which reads and writes D1 and returns JSON. In development this is two processes (Wrangler on `:8787`, Vite on `:5173`); in production it is a single deployed Worker.

---

## Getting started

**Prerequisites:** Node.js 20+, a Cloudflare account (only for deploying), and a Gemini or Anthropic API key for the assistant.

```bash
npm install

# create .dev.vars (gitignored) with your key
#   GEMINI_API_KEY=...
#   CHATBOT_ENABLED=true

npm run dev          # API on http://localhost:8787, web on http://localhost:5173
npm run typecheck    # TypeScript for the app and the worker
npm run build        # typecheck + production build into dist/
```

Open **http://localhost:5173**. The D1 schema is created idempotently on first request (`ensureReady` in `worker/db.ts`), so there is no migration step for local development.

### Environment variables

| Variable | Required | Purpose |
| --- | --- | --- |
| `GEMINI_API_KEY` | one of these | Google AI Studio key for the assistant |
| `ANTHROPIC_API_KEY` | one of these | Takes priority over Gemini when set |
| `CHATBOT_ENABLED` | no | `false` hides the widget and disables `/api/chatbot` |
| `CHATBOT_MODEL` | no | Overrides the default model |

Copy `.env.example` to `.dev.vars` for local work. Never commit real values — `.dev.vars`, `.env`, `dist`, `node_modules` and `.wrangler` are ignored.

### Deploy

```bash
npx wrangler deploy
npx wrangler secret put GEMINI_API_KEY   # keep secrets out of wrangler.jsonc
```

---

## Project layout

```
src/            React SPA — pages (deck, liked, requests, matches, compare, profile, admin),
                components (match cards, score breakdown, viewing scheduler, chatbot), lib/
worker/         Hono API — index.ts (routes + auth), db.ts (schema + queries),
                engine.ts + filters.ts (matching), auth.ts, chatbot*.ts (assistant),
                applicationFile.ts (rental application PDF)
shared/         Types and constants used by both sides (areas, hobbies, scores)
scripts/        Build helpers (PDF embedding)
```

### API at a glance

`POST /api/signup`, `POST /api/login`, `DELETE /api/session`, `GET|PUT|DELETE /api/me`, `POST /api/me/listing`, `GET /api/recommendations`, `POST /api/interest`, `GET /api/liked`, `GET /api/requests`, `GET /api/matches`, `POST /api/matches/:id/{request,withdraw,screening,accept,sign,rate}`, viewing routes under `/api/matches/:id/viewing`, `GET /api/matches/:id/application.pdf`, `GET /api/notifications`, `GET /api/trust/:id`, `POST /api/reports`, admin routes under `/api/admin/*`, and the assistant at `POST /api/chatbot`.

---

## Notes

- Every `/api/*` route except signup and login resolves the user from the session cookie and scopes queries to that user; unauthenticated calls return **401**.
- The assistant is additive: it reads through existing database helpers, adds no tables or columns, and never returns API keys.
- No test suite is included; verification is `npm run typecheck`, `npm run build`, and manual browser checks.
