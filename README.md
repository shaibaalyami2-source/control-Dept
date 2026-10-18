# CRM – Engineering & Maintenance Administration

Arabic (RTL) incident / work-order CRM on Supabase, refactored from one 1.3 MB HTML file into a
structured project, with a built-in **AI assistant (Grok, Claude or Gemini)** available on every screen.

## Run it

```bash
cp .env.example .env        # then fill GEMINI_API_KEY
node server/index.js        # http://localhost:3000   (Node >= 18.17, no npm install needed)
npm test                    # backend tests (mock Supabase + mock Gemini)
```

The server serves `public/`, the AI route and one admin-only data route. Deploy it anywhere that
runs Node (Render, Railway, Fly, a VPS). If you host the front-end elsewhere, set `API_BASE` in
`public/js/core/config.js` to the server's URL and list the front-end origin in `ALLOWED_ORIGINS`.

### Environment variables

| Variable                                   | Purpose                                                                                     |
| ------------------------------------------ | ------------------------------------------------------------------------------------------- |
| `ANTHROPIC_API_KEY`                        | Claude key from console.anthropic.com. When set, Claude answers the chat.                   |
| `ANTHROPIC_MODEL`                          | Claude model id (default `claude-haiku-5-5`: fast and inexpensive).                         |
| `AI_PROVIDER`                              | `anthropic` or `gemini` to force one; otherwise Claude if its key exists, else Gemini.      |
| `GEMINI_API_KEY`                           | Google AI key. **Server only.** Without it the chat answers "not enabled".                  |
| `GEMINI_MODEL`                             | Model id (default in `.env.example`; change it to any Gemini model you have access to).     |
| `GEMINI_MAX_OUTPUT_TOKENS`                 | Cap per reply.                                                                              |
| `GEMINI_API`                               | `interactions` (default; the newest models only work with it) or `generate` (older models). |
| `GEMINI_THINKING_LEVEL`                    | `minimal`/`low`/`medium`/`high`. Default `low` keeps replies fast.                          |
| `SUPABASE_URL`, `SUPABASE_PUBLISHABLE_KEY` | Used to verify the caller's session.                                                        |
| `ALLOWED_ORIGINS`                          | Extra CORS origins (comma separated). Same-origin needs nothing.                            |
| `AI_RATE_PER_MINUTE`, `AI_RATE_PER_DAY`    | Per-user limits.                                                                            |

## The AI assistant

- **Backend** – `POST /api/gemini` (`server/routes/gemini.js`). It verifies the Supabase session
  (valid JWT **and** an active row in `profiles`), rate-limits per user, caps concurrency and body
  size, builds the system prompt on the server (the browser can only pick a screen id from a fixed
  list), streams the answer as SSE, aborts upstream when the user closes the tab, and returns only
  generic errors – upstream details and the key never reach the browser.
- **Frontend** – `public/js/ai/` (`services.js` per-screen prompts and context, `chat.js`
  state/streaming, `widget.js` UI) and `public/css/05-ai-chat.css`. A floating ✨ button on every
  screen opens the panel; it follows the active tab, offers quick prompts for that screen, supports
  Arabic/English, dark mode, Stop, New chat and Copy. History lives in `sessionStorage` and is
  wiped on logout.
- **Privacy** – the optional screen context contains **counts only** (totals, by status/priority,
  overdue). Names, employee numbers and phone numbers are never sent; the user can switch the
  context off. The assistant cannot read the database or press buttons.

## iPhone / Android app (PWA) and notifications

The site is an installable web app: `public/manifest.webmanifest`, `public/sw.js` (offline shell +
push), icons in `public/icons/`, phone layout in `public/css/06-mobile.css`, install/push logic in
`public/js/pwa.js`.

**Install on iPhone:** open the site in **Safari** → Share → **Add to Home Screen** → open it from the new
icon. (iOS 16.4+. Notifications only work for the installed app, not for a Safari tab.)

**Turn notifications on:** open the installed app, tap the banner or ☰ → 🔔 _تفعيل الإشعارات_, allow.

### One-time server setup for push

1. `node scripts/generate-vapid.js` → copy the four printed values into Render → Environment
   (`VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT`, `PUSH_WEBHOOK_SECRET`).
2. Supabase → Project Settings → API → copy the **service_role** key into Render as
   `SUPABASE_SERVICE_ROLE_KEY` (secret; server only).
3. Supabase → SQL Editor → run `supabase/push.sql` (creates the `push_subscriptions` table, locked by RLS).
4. Supabase → Database → **Webhooks** → create one webhook per table, event **Insert**, type _HTTP Request_,
   method POST, URL `https://YOUR-SITE.onrender.com/api/push/hook`, header `x-webhook-secret: <PUSH_WEBHOOK_SECRET>`:
   - `notifications` (alerts addressed to one user), `internal_messages` (team/private messages),
     `employee_notes` (to admins), and `reports` (new report → `PUSH_REPORT_ROLES`).
   - If the database already creates a `notifications` row for every new report, skip the `reports`
     webhook, otherwise people get the alert twice.

Server routes: `GET /api/push/key`, `POST /api/push/subscribe` (signed-in users), `POST /api/push/unsubscribe`,
`POST /api/push/hook` (secret header). Messages are end-to-end encrypted (RFC 8291) and signed (VAPID); the
encryption is verified against the RFC test vector in `test/push.test.js`.

## Layout

```
public/            static front-end (classic scripts, shared global scope)
  js/core/         config, supabase client, state, i18n, ui, permissions, data, auth, realtime, router, shell
  js/features/     one file per screen/feature
  js/ai/           AI assistant
  css/             01-base … 05-ai-chat (cascade order preserved from the original)
server/            zero-dependency Node server (AI route, admin seed route, static files)
data/              employees-seed.json (served only to admins by the server – not in public/)
legacy/            original code that was dead in the running app (not loaded)
test/              backend tests
```

Scripts are still classic scripts (not ES modules) because the markup uses inline `onclick="fn()"`
handlers that need global functions. Load order is in `public/index.html`.

## What changed and what was fixed

- Seven stacked "patch" scripts and four injected style blocks were merged into the owning
  modules; duplicate and dead definitions removed (236 provably overridden CSS declarations dropped).
  Every screen was compared with the original in a headless browser (DOM and pixels); the
  remaining differences are the intended fixes below.
- **Supervisor permission bug** – `can()` now falls back to the supervisor's own permissions
  instead of always denying; creating a report also checks `create_report`.
- **`esc()` XSS fix** – quotes were not escaped, so values inside attributes could break out.
- **Double submit** – report creation is guarded and idempotent (RPC with a request key).
- **Single boot/logout path** – login and session-restore used to diverge (different realtime
  channels, permissions loaded in one only). One channel now carries all subscriptions.
- **Reverse translation** – switching back to Arabic no longer rewrites brand strings (CRM, QA).
- **Toasts** honour the error flag; the admin password-requests screen is reachable again
  (button in the user directory).
- **Secrets** – the 3,167-row employee list is no longer shipped to every browser: it is fetched
  from an admin-only endpoint (so employee sync now needs this Node server). The hard-coded temporary
  supervisor password was removed from the front-end.

## Things you should still do

1. **Rotate the temporary supervisor password** that was in the original file – it was public to
   anyone who could load the page.
2. **Check Row Level Security** on every Supabase table. The publishable key is public by design;
   RLS is the real access control, and the original relied on front-end checks.
3. Add a **Content-Security-Policy** at your host/CDN (the app uses inline handlers, so it needs
   `'unsafe-inline'` for scripts until those are migrated).
4. Pin the two CDN scripts (supabase-js, qrcodejs) to exact versions or self-host them.
5. If the AI key was ever pasted anywhere public, create a new one.
6. `legacy/crm-feature-v13.original.js` holds a monthly schedule and typo-analysis feature that the
   original never actually displayed (later patches overrode it). It is not loaded.
