# TaskFlow

A full-stack team task manager with built-in **Docker AI analysis**.

**Stack:** React 18 + Vite (frontend) · Node.js + Express (API) · MongoDB + Mongoose (database)

## Features

| Area | What you get |
|---|---|
| **Authentication** | Register / sign in, bcrypt password hashing, JWT sessions, profile update, change password, rate-limited auth endpoints |
| **Tasks** | Create, edit, delete; status (To do / In progress / In review / Done), priority, due dates, labels, assignee, project; search, filters, sorting; **list view and drag-and-drop Kanban board** |
| **Task detail** | Comments, file attachments (download / delete), full activity history |
| **Team collaboration** | Shared workspaces; roles **owner / admin / member / viewer** enforced on the server; invite by email (single-use, expiring link **or** in-app accept/decline); change roles, remove members, leave workspace |
| **Projects** | Per-workspace projects with colour, progress bar, archive and delete |
| **Notifications** | In-app notifications for assignments, status changes, comments, invitations, role changes; unread badge (auto-refreshing), bell dropdown, notifications page |
| **Dashboard** | Totals, overdue, due today, done this week, completion rate, status and priority breakdown, upcoming deadlines, recent activity |
| **Docker AI analysis** | Paste or upload a **Dockerfile, docker-compose file or container/build logs**. A built-in rule engine (40+ checks) finds security and reliability problems with line numbers and fixes; an optional LLM adds explanations. Secrets are **redacted before** anything is sent to an AI model. History, downloadable reports, one-click "create task" from a finding |
| **UX** | Responsive layout, light / dark / system theme, toasts, keyboard-accessible controls |

## Quick start with Docker (recommended)

```bash
cp .env.example .env
# edit .env and set JWT_SECRET  (openssl rand -hex 32)
docker compose up --build
```

Open <http://localhost:8080> and create an account. MongoDB data and uploaded files live in Docker volumes.

## Run locally for development

Requires Node.js 20+ and a MongoDB instance (local, or free [MongoDB Atlas](https://www.mongodb.com/atlas)).

```bash
npm run install:all
cp backend/.env.example backend/.env      # set MONGODB_URI and JWT_SECRET
npm run dev:backend                       # API on http://localhost:5000
npm run dev:frontend                      # App on http://localhost:5173 (proxies /api)
```

Quick MongoDB with Docker: `docker run -d -p 27017:27017 --name tf-mongo mongo:7`

Run the unit tests: `npm test`

## Enabling the AI model (optional)

The Docker analyzer works out of the box with the rule engine. To add LLM insights, point it at any
OpenAI-compatible chat-completions endpoint in `backend/.env` (or `.env` for Docker):

```env
# Ollama (free, local)
AI_API_URL=http://127.0.0.1:11434/v1/chat/completions   # in Docker: http://host.docker.internal:11434/...
AI_API_KEY=ollama
AI_MODEL=llama3.2:3b
```

If the model is unreachable the app falls back to the rule engine and says so in the report.

## Team invitations

Admins create an invitation for an email address and receive a **link to share** (SMTP email sending is not built in).
The link works only after the invited person signs in / registers with that same email. If they already have an
account they also see the invitation under **Team** and can accept or decline in-app.

## Project layout

```
backend/src
  app.js server.js config.js db.js
  models/        User Workspace Project Task Notification Invitation Analysis
  routes/        auth workspaces invitations projects tasks notifications analysis dashboard
  services/      access (role checks) notify dockerAnalyzer aiProvider files
  middleware/    auth error
backend/test     analyzer unit tests
frontend/src
  pages/         Login Dashboard Tasks Projects Team Notifications Analyzer Settings InviteAccept
  components/    Layout TaskModal TaskViews ui Icon
  lib/           api AuthContext AppContext format
```

## REST API (all under `/api`, JWT bearer auth except register/login/health)

`POST /auth/register · POST /auth/login · GET|PATCH /auth/me · POST /auth/change-password`
`GET|POST /workspaces · GET|PATCH|DELETE /workspaces/:id · PATCH|DELETE /workspaces/:id/members/:userId`
`GET|POST /workspaces/:id/invitations · DELETE /workspaces/:id/invitations/:invId`
`GET /invitations/mine · POST /invitations/accept · POST /invitations/:id/accept|decline`
`GET|POST /projects · PATCH|DELETE /projects/:id`
`GET|POST /tasks · GET|PATCH|DELETE /tasks/:id · POST|DELETE /tasks/:id/comments[/:cid] · POST|GET|DELETE /tasks/:id/attachments[/:aid]`
`GET /notifications · GET /notifications/unread-count · POST /notifications/read-all · PATCH /notifications/:id/read · DELETE /notifications/:id`
`GET /dashboard · GET|POST /analysis · GET|DELETE /analysis/:id · GET /analysis/status · GET /health`

## Security notes

- Passwords hashed with bcrypt (cost 12); JWT signed HS256 with a secret you provide; the server refuses to start with a weak or default `JWT_SECRET`.
- Helmet headers, CORS allow-list, rate limits (global, auth, and analyzer), Zod input validation, regex-escaped search.
- Authorization is enforced server-side per workspace role; invitation tokens are stored hashed.
- Uploads are stored outside the web root, renamed to random names, size-limited, script/executable types blocked, and served only to authorised members as downloads with `nosniff`.
- Before a public deployment: put it behind HTTPS, restrict MongoDB network access, set exact `CORS_ORIGIN`, back up the database, and consider S3-style storage for uploads if you run several API instances.

## Known limitations

- No email delivery (invitation links are copied manually); notifications are in-app only and refresh every 30 s (no websockets).
- The analyzer reads text only (no PDF/image OCR).
- Uploaded files use local disk storage (single-instance).
