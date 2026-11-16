# Edvisery CRM v2 - Phase 1 (foundation)

React + TypeScript + Tailwind frontend, Node + Express + TypeScript API, PostgreSQL.
Monorepo (npm workspaces):

    packages/shared   call-flow logic used by BOTH web and API (validation, auto tasks, stage rules)
    apps/api          REST API, SQL migrations, auth, audit log
    apps/web          SPA: login, leads, lead details, call modal, call-flow settings
    deploy/           PM2 + Nginx examples

## Run locally

Needs Node 20+ and Docker (or any PostgreSQL 13+).

    npm install
    docker compose up -d db
    cp .env.example apps/api/.env      # edit JWT_SECRET and SEED_ADMIN_PASSWORD
    npm run migrate
    npm run seed
    npm run dev:api                    # http://localhost:4000/health
    npm run dev:web                    # http://localhost:5173  (proxies /api to :4000)
    npm test                           # shared logic tests

Log in with the SEED_ADMIN_EMAIL / SEED_ADMIN_PASSWORD you set in `.env`.

## What Phase 1 includes

- Login (JWT, bcrypt), roles: admin, manager, counsellor. Counsellors only see their own leads.
- Leads: list with search, details, edit, soft delete (manager/admin).
- Call flow from the database. Admin edits it in **Call flow settings**; no code change or redeploy.
- Saving an activity runs in ONE transaction: activity + auto tasks/notes + stage change + document records + audit entry.
- The server re-validates every call with the same shared rules (past dates, required date/time/reason), so the UI cannot be bypassed.
- Tasks tab with done/undo. Documents tab (metadata only, see below).
- Structured JSON logs (pino), `/health`, helmet, CORS, audit_log table.

## Not in Phase 1 (planned)

- Realtime (Socket.IO), Redis/BullMQ jobs (reminders when a task falls due, imports, campaigns).
- Resend email + webhooks, telephony adapter (Exotel), WhatsApp.
- File upload: only the file name and size are saved now. `documents.storage_key` is ready for private object storage.
- Rate limiting on login, password reset, user management screen.
- Automated API/DB integration tests. The shared logic has unit tests; the API and SQL have not been run against a live database yet, so run the steps above and report anything that fails.

## Deploy on the Hostinger VPS (separate directory, process, database, subdomain)

1. Create a database and user in PostgreSQL. Put the values in `apps/api/.env` (NODE_ENV=production, long random JWT_SECRET, CORS_ORIGIN=https://your-subdomain).
2. `npm ci && npm run build && npm run migrate && npm run seed`
3. `pm2 start deploy/ecosystem.config.cjs && pm2 save`
4. Copy `apps/web/dist/*` to the Nginx web root and use `deploy/nginx.conf.example`; get HTTPS with certbot.
5. Point your uptime monitor at `https://your-subdomain/health`.
