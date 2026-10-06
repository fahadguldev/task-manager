# Task Manager (TaskAI)

A task manager you operate from WhatsApp: send a message, get an AI-parsed task, and let scheduled jobs fire reminders and recurring-task recurrences — with a React dashboard for everything the chat interface cannot show.

- **Live App:** https://task-ai-tau.vercel.app
- **Custom domain:** https://taskai.studio
- **Repository:** https://github.com/fahadguldev/task-manager

## Project Overview

| Directory | Package | Role |
| --- | --- | --- |
| `backend/` | `taskmanager-backend` | Express + Mongoose API, WhatsApp bot, cron jobs |
| `frontend/` | React SPA | Vite + React + Tailwind dashboard |
| `docker-compose.yml` | 3 services | frontend (80), backend (5000), mongo |

The product's distinguishing idea is that WhatsApp *is* the input surface. The bot parses natural messages into structured tasks; the dashboard is for review, analytics, and administration.

## Problem & Solution

Task apps fail on capture friction — opening an app to log a task is the step people skip. Since everyone already has WhatsApp open, capturing there removes that step entirely.

Flow: message arrives → `messageHandler.js` interprets it → an AI suggestion layer (`aiSuggestions`) structures it → a `Task` row is created → `reminder.job.js` and `recurringTask.job.js` push follow-ups back out over WhatsApp.

## Tech Stack

| Layer | Technology |
| --- | --- |
| Frontend | React, React Router, Tailwind CSS (`@tailwindcss/vite`), Lucide React |
| State & Data | Axios, React context (`src/context`) |
| Backend | Express, `cors`, `cookie-parser`, `nodemon` |
| Database | MongoDB (Mongoose) — `task.model.js`, `user.model.js` |
| Auth & Security | `jsonwebtoken`, `bcrypt`, `middleware/auth.js`, `middleware/authorizeRole.js` |
| WhatsApp | `whatsapp-web.js`, `puppeteer`, `qrcode-terminal` |
| Scheduling | `node-cron` (`jobs/reminder.job.js`, `jobs/recurringTask.job.js`) |
| AI | suggestion service (`aiSuggestions.routes.js`, `services/`) |
| Tooling | Vite, ESLint, Nodemon |
| Infrastructure | Dockerfile (2 services), `docker-compose.yml`, GitHub Actions (`deploy.yml`), `vercel.json` |

## System Architecture

```
   WhatsApp phone  <-->  whatsapp-web.js (Puppeteer session)
                              |
                     services/whatsapp/
                       client.js      (WA Web client lifecycle)
                       queue.js       (outbound message queue)
                       messageHandler.js  (inbound parse)
                       state.js       (conversation state)
                       menu.js        (menu-driven flows)
                              |
        whatsappBot/  taskScheduler.js, whatsappBot.controller.js,
                     whatsapp.route.js
                              |
   +--------------------------+--------------------------+
   | backend (Express, :5000) |                          |
   |  routes/  task, user, admin, whatsapp, aiSuggestions|
   |  controllers/             |  middleware/ auth,      |
   |  jobs/    reminder.job.js |            authorizeRole,|
   |           recurringTask.job.js  errorHandler        |
   |  services/ whatsapp/, recurringTask.service.js      |
   +--------------------------+--------------------------+
                              |            |
                       [ MongoDB ]   node-cron ticks
                              |
   frontend (React, Vite, :5173 -> :80 in Docker)
     pages/  Login SignUp Home Dashboard AddTodo
             Summary Analytics Profile AdminPanel
```

Docker wiring: `frontend` (host 80 → container 5173) depends on `backend` (host 5000); `backend` depends on `mongo`. The backend mounts named volumes **`.wwebjs_auth` and `.wwebjs_cache`** so the WhatsApp login survives container restarts, and runs as `${UID:-1000}:${GID:-1000}` to keep those files writable on the host.

## Key Features

- **WhatsApp capture** — natural-language messages become tasks through `messageHandler.js`.
- **Menu-driven chat flows** — `services/whatsapp/menu.service.js` for structured choices when free text is ambiguous.
- **Outbound queue** — `services/whatsapp/queue.js` serializes sends so reminders never trip WhatsApp's rate limits.
- **Reminders** — `jobs/reminder.job.js` on `node-cron` fires upcoming-task notifications.
- **Recurring tasks** — `jobs/recurringTask.job.js` + `services/recurringTask.service.js` roll a completed occurrence forward.
- **AI task suggestions** — `aiSuggestions.routes.js` proposes structured task fields from raw text.
- **Dashboard & analytics** — `Dashboard`, `Summary`, `Analytics` pages with charts.
- **Task CRUD** — `AddTodo`, `task.routes.js`, `task.model.js`.
- **Auth** — JWT + bcrypt, with `authorizeRole.js` for role-gated admin routes.
- **Admin panel** — `AdminPanel.jsx` + `admin.routes.js` for user and system administration.
- **Profile** — `Profile.jsx` + `user.routes.js`.
- **QR login** — `qrcode-terminal` prints the pairing code for WhatsApp Web.
- **Health check on boot** — `npm start` runs `node cleanup.js && nodemon index.js`, clearing stale session state first.
- **Docker one-liner** — `docker compose up` starts all three services.
- **CI/CD** — `.github/workflows/deploy.yml`.

## Setup & Run

Local (two terminals):

```bash
git clone https://github.com/fahadguldev/task-manager
cd task-manager/backend
npm install
npm start           # node cleanup.js && nodemon index.js  -> :5000
# scan the QR code printed in the terminal with WhatsApp

cd ../frontend
npm install
npm run dev         # Vite -> :5173
```

Docker (recommended; persists the WhatsApp session):

```bash
cd task-manager
docker compose up --build
# frontend http://localhost   backend :5000   mongo (internal)
```

Environment: `backend/.env` with the MongoDB URI, JWT secret, and any AI suggestion provider key. `docker-compose.yml` reads it via `env_file: ./backend/.env`.

## Technical Decisions

- **`whatsapp-web.js` instead of the WhatsApp Business API.** No Meta approval, no per-conversation pricing — paid for with a Puppeteer session that must be kept alive, hence `cleanup.js` on start and the `.wwebjs_auth` volume.
- **Cron jobs in-process rather than a separate scheduler service.** Two jobs on a single box is well within `node-cron`'s comfort zone; a distributed scheduler would add failure modes with no benefit.
- **A dedicated outbound queue.** WhatsApp rejects bursts. `queue.js` paces sends and decouples "job decided to notify" from "message left the phone."
- **Roles as middleware, not inline checks.** `authorizeRole.js` keeps the permission matrix in one place rather than scattered through controllers.
- **`errorHandler.js` as the last Express middleware** so controllers stay free of repetitive try/catch.
- **Session cleanup as a start script.** Running `cleanup.js` before `nodemon` means a crashed or half-authenticated Puppeteer session never blocks boot.
- **Volume-persisted session + fixed UID/GID** so Docker restarts do not force a re-scan.

## Challenges & Solutions

- **WhatsApp Web sessions do not survive restarts.** Solved with `.wwebjs_auth` / `.wwebjs_cache` bind mounts in `docker-compose.yml` — the login persists across container recreation.
- **Container/host file ownership.** The backend runs as `${UID:-1000}:${GID:-1000}` so session files written by the container remain editable by the host user; `fix-permissions.sh` handles the case where they do not.
- **Stale session state on redeploy.** `cleanup.js` runs as part of `npm start` to reset cached session files before the bot boots.
- **Cron jobs firing duplicate reminders.** Reminder state lives in MongoDB and is re-read each tick, so a restart does not re-fire already-sent notifications.
- **Two very different UIs from one codebase.** Chat is stateless-ish and menu-driven; the dashboard is forms and charts — kept apart by the `services/whatsapp` vs. `pages/` split.

## Honest Gaps

- **Committed third-party token.** A Wit.ai token is present in the repository's configuration. It is **not** reproduced here; treat it as compromised and rotate it in the provider console before any reuse of this code.
- **Placeholder analytics figures.** Some analytics content is hardcoded/fabricated rather than computed from live task data — do not read `Analytics` numbers as production metrics.
- **`test` script is a no-op.** `backend/package.json` has `test=echo "Error: no test specified" && exit 1`; there is no test suite despite the scheduler and bot logic being exactly the kind of code that needs one.
- **No `.env.example`.** Required variables must be inferred from `config/` and `docker-compose.yml`.
- **Whitescreen risk noted in `DOCKER_SETUP.md`.** The repo ships troubleshooting docs for dev-mode polling issues (`VITE_CHOKIDAR_USEPOLLING=true`), which suggests the Docker dev loop was painful rather than smooth.
- **Repository history is a single squashed commit**, so no development timeline can be inferred.

## Deployment Status

| Surface | URL | Status |
| --- | --- | --- |
| App | https://task-ai-tau.vercel.app | live |
| Custom domain | https://taskai.studio | live |
| Backend / MongoDB / WhatsApp worker | — | not on Vercel (needs a persistent host for Puppeteer + cron) |

## Lessons Learned

- Removing capture friction beats adding features: putting task entry inside an app people already have open is the whole product.
- Anything that automates a consumer messaging app needs three things people forget: a human bootstrap (QR), durable session storage, and paced outbound sending.
- Cron jobs must be idempotent — after a restart they will run again, and the database is the only safe memory of what already happened.
- Committed credentials in a portfolio project are worse than no project: rotate them, and never treat a leaked token as an implementation detail.
- A `cleanup` script in front of the bot boot is cheap insurance against the single most common WhatsApp-automation failure (half-restored sessions).

## Author

**Muhammad Fahad** - [@fahadguldev](https://github.com/fahadguldev)

Repository: https://github.com/fahadguldev/task-manager
