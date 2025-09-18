# Technical Design Document (TDD) — Workshop Companion App

**Version:** v0.3 • **Date:** 18 Sep 2025 • **Owner:** PB\&S

---

## 0) Scope

MVP implementation for a workshop companion web app.

- **Frontend:** Next.js **Pages Router**, static export, deployed to **Heroku**
- **Backend:** FastAPI on **Heroku**
- **DB:** Supabase Postgres
- **Sync:** REST + polling (3s; pause when tab backgrounded)
- **Auth:** Participant (code+name), Organizer (demo login, hashed pwd)
- **State:** RTK Query for server state; minimal local state for UI

---

## 1) Architecture Overview

- **Frontend**: Next.js SPA with Pages Router. State via RTK Query. Polling every 3s for state/modules, 15s for reaction aggregates.
- **Backend**: FastAPI app. Service layer with business logic, repository layer for DB access. Organizer session via cookie, participant identity via UUID.
- **Database**: Supabase Postgres. Tables: organizer_accounts, workshops, modules, resources, participants, organizers, state.
- **Deployment**: Both frontend and backend hosted on Heroku free dynos.
- **Observability**: Logs, health checks. No advanced metrics in MVP.

---

## 2) Backend (FastAPI)

### 2.1 Project Structure

```
backend/
  app/
    api/routers/
      public.py
      organizer.py
    core/
      config.py
      security.py
      rate_limit.py
    services/
      workshops.py
      modules.py
      participants.py
      reactions.py
      organizers.py
      state.py
    repositories/
      workshops.py
      modules.py
      participants.py
      organizers.py
      resources.py
      state.py
    models/
      (SQLAlchemy models)
    schemas/
      (Pydantic request/response DTOs)
    main.py
  alembic/
    versions/
```

### 2.2 Data Model (high-level)

- **organizer_accounts**: id, login_id, password_hash, created_at
- **workshops**: id, code, title, description, agenda_json, start_at, status, primary_controller_id, created_by, created_at
- **modules**: id, workshop_id, index, title, content_markdown, created_at
- **resources**: id, workshop_id, name, url, type(link/pdf/video/image), created_at
- **participants**: id, workshop_id, name, email, joined_at, present
- **organizers**: id, workshop_id, user_id, role, added_at
- **state**: workshop_id, current_step_index, elapsed_ms, updated_at

### 2.3 API Endpoints (skeleton)

- **Public (Participant)**

  - POST /join
  - GET /w/{code}/state
  - GET /w/{code}/modules
  - GET /w/{code}/resources
  - POST /w/{code}/react

- **Organizer (Auth cookie)**

  - POST /organizer/login
  - POST /o/workshops
  - POST /o/workshops/{id}/publish
  - POST /o/workshops/{id}/start
  - POST /o/workshops/{id}/end
  - POST /o/workshops/{id}/step
  - POST /o/workshops/{id}/modules
  - PATCH /o/workshops/{id}/modules/reorder
  - GET /o/workshops/{id}/participants
  - POST /o/workshops/{id}/takeover

### 2.4 Technical Standards

- Organizer cookie: httpOnly, signed, 8h max-age, idle-timeout 30m
- Participant token: opaque UUID, stored client-side
- Rate limits: reactions (1/15s, 3/min burst), joins (10/min/IP)
- Errors: standardized JSON { code, message, details? }
- Schema migrations: Alembic; demo organizers seeded

### 2.5 Deployment (Heroku)

- Procfile: `web: uvicorn app.main:app --host 0.0.0.0 --port $PORT`
- Buildpacks: heroku/python
- Env vars: DB URL, COOKIE_SECRET, RATE_LIMITS, CORS_ORIGIN

---

## 3) Frontend (Next.js Pages Router)

### 3.1 Project Structure

```
frontend/
  src/
    pages/
      index.tsx
      o/
        login.tsx
        dashboard.tsx
      s/
        [code].tsx
    lib/api/
      (RTK Query slices)
    components/
      JoinForm.tsx
      OrganizerLoginForm.tsx
      ModuleViewer.tsx
      ProgressBar.tsx
      NavArrows.tsx
      BackToCurrent.tsx
      EmojiTray.tsx
      ResourcesList.tsx
      EndedStatusPage.tsx
    store/
      index.ts
    styles/
```

### 3.2 State & Networking

- RTK Query slices for participant + organizer APIs
- Polling intervals: 3s for session state/modules, 15s for reactions
- Participant identity: stored in localStorage
- Organizer: session cookie

### 3.3 UI Standards

- Tailwind + shadcn/ui components
- Markdown rendering via react-markdown + rehype-sanitize
- Mobile-first layout; modules sized to avoid vertical scroll

### 3.4 Error Handling

- API errors surfaced via toasts; standardized error codes

### 3.5 Deployment (Heroku)

- Build: `next build && next export` → `out/`
- Serve with Node static server
- Procfile: `web: node server.js`
- Buildpacks: heroku/nodejs

---

## 4) Testing Strategy

- Backend: pytest unit + API contract tests
- Frontend: Jest/RTL for RTK Query hooks + components
- E2E smoke flow: join → publish → start → step → react → end
- Load test: simulate 500 participants + reaction bursts

---

## 5) Next Steps

- Define DTO schemas for each API endpoint
- Prepare Alembic migration scripts
- Scaffold FastAPI routers & services
- Scaffold Next.js pages, RTK slices, and store
- Create Procfiles and Heroku pipelines for frontend & backend

## 9) Backend — Detailed (No Code)

### 9.1 Folder Structure (FastAPI)

- `app/`

  - `api/routers/` (public, organizer)
  - `core/` (config, security, rate_limit, middleware)
  - `services/` (workshops, modules, participants, reactions, organizers, state)
  - `repositories/` (workshops, modules, participants, organizers, resources, state)
  - `models/` (sqlalchemy models)
  - `schemas/` (pydantic DTOs)
  - `main.py`

- `alembic/` (migrations)
- `Procfile`, `requirements.txt`, `runtime.txt`

### 9.2 Data Models (Fields)

- **organizer_accounts**: id (uuid), login_id (text, unique), password_hash (text), created_at (timestamptz)
- **workshops**: id (uuid), code (char(4), unique, A–Z+digits excl. 0/1/O/I), title (text), description (text), agenda_json (jsonb), start_at (timestamptz), status (enum: draft/published/live/ended), primary_controller_id (uuid, nullable), created_by (uuid), created_at (timestamptz)
- **modules**: id (uuid), workshop_id (uuid fk), index (int), title (text), content_markdown (text), created_at (timestamptz)
- **resources**: id (uuid), workshop_id (uuid fk), name (text), url (text), type (enum: link/pdf/video/image), created_at (timestamptz)
- **participants**: id (uuid), workshop_id (uuid fk), name (text), email (text, nullable), joined_at (timestamptz), present (bool)
- **organizers**: id (uuid), workshop_id (uuid fk), user_id (uuid fk organizer_accounts), role (enum: organizer/controller), added_at (timestamptz)
- **state**: workshop_id (uuid pk/fk), current_step_index (int), elapsed_ms (bigint), updated_at (timestamptz)
- **reaction_counters (ephemeral)**: in-memory only (workshop_id, module_id, emoji → count, updated_at)

### 9.3 DTOs (Requests/Responses)

- **Auth**

  - OrganizerLoginRequest { loginId, password }
  - OrganizerLoginResponse { ok, organizerId }
  - JoinRequest { code, name, email? }
  - JoinResponse { participantToken, workshop: { id, code, title, totalSteps }, state }

- **State/Modules/Resources**

  - GetStateResponse { currentStep, agenda, elapsedMs, upcoming }
  - GetModulesResponse \[ { id, index, title, contentMarkdown } ]
  - GetResourcesResponse \[ { id, name, url, type } ]

- **Control**

  - PublishWorkshopRequest { workshopId }
  - StartWorkshopRequest { workshopId }
  - EndWorkshopRequest { workshopId }
  - StepRequest { action: next|prev|goto, index? }
  - TakeoverRequest { workshopId }
  - ParticipantsListResponse \[ { participantId, name, email? } ]

- **Reactions**

  - ReactionRequest { emoji }
  - ReactionResponse { ok }

### 9.4 API Standards

- JSON only; snake_case in DB, camelCase in API
- Status codes: 200/201/204; 400/401/403/404/409/429
- Error envelope: { code, message, details? } with codes: INVALID_CODE, UNAUTHORIZED, FORBIDDEN, NOT_PRIMARY, CONFLICT, RATE_LIMITED, NOT_FOUND, VALIDATION_ERROR
- Caching: ETag on state/modules responses; support If-None-Match

### 9.5 Security & Limits

- Organizer cookie: httpOnly, secure, signed; max-age 8h; idle-timeout 30m; manual logout
- Participant token: opaque UUID stored client-side
- Rate limits: reactions 1/15s + 3/min burst; joins 10/min/IP; organizer login 10/min/IP
- Sanitization: markdown sanitized server-side and client-side

### 9.6 Deployment (Heroku — Backend)

- Buildpack: heroku/python
- Procfile: `web: uvicorn app.main:app --host 0.0.0.0 --port $PORT`
- Env vars: `DATABASE_URL`, `CORS_ORIGIN`, `COOKIE_SECRET`, `RATE_LIMITS`, `LOG_LEVEL`
- DB: Supabase connection via `DATABASE_URL`
- Migrations: Alembic run on release phase
- Health: `/healthz`

---

## 10) Frontend — Detailed (No Code)

### 10.1 Folder Structure (Next.js Pages + RTK Query)

- `src/`

  - `pages/`

    - `index.tsx` (landing: participant join / organizer CTA)
    - `o/login.tsx` (organizer login)
    - `o/dashboard.tsx` (organizer dashboard)
    - `s/[code].tsx` (participant session)

  - `components/`

    - `JoinForm.tsx`, `OrganizerLoginForm.tsx`
    - `ModuleViewer.tsx` (markdown), `ProgressBar.tsx`
    - `NavArrows.tsx`, `BackToCurrent.tsx`, `EmojiTray.tsx`
    - `ResourcesList.tsx`, `EndedStatus.tsx`

  - `lib/api/` (RTK Query slices)

    - `sessionApi.ts` (state/modules/resources/react)
    - `organizerApi.ts` (login, workshops, control, participants)

  - `store/` (Redux store + RTK Query config)
  - `styles/` (Tailwind config and globals)
  - `utils/` (visibility, polling control, storage)
  - `config/` (env, constants, error codes)

### 10.2 Standards & Rules

- **State**: RTK Query for server state; UI state local where trivial
- **Polling**: 3s for state/modules; 15s for reaction aggregates; pause on `document.hidden`
- **Identity**: participant token in localStorage; organizer via httpOnly cookie
- **Markdown**: `react-markdown` + `rehype-sanitize` (allow links, lists, headings)
- **Accessibility**: keyboard nav for arrows; ARIA for progress
- **Errors**: render standardized JSON error messages; toasts for non-blocking issues

### 10.3 Deployment (Heroku — Frontend)

- Build: `next build && next export` → `out/`
- Serve: Node static server (cache immutable assets)
- Procfile: `web: node server.js`
- Env vars: `API_BASE_URL`, `SENTRY_DSN` (optional), `APP_ENV`
- Health: `/healthz` static endpoint

---

## 11) Non-Functional Requirements

- **Performance**: initial load < 2s on mid-tier 4G; module view interactions < 100ms
- **Reliability**: graceful polling resume after network loss
- **Scalability**: up to 500 concurrent participants per workshop (polling-based)
- **Security**: basic OWASP A1–A3 hygiene; sanitized markdown; cookie protections

---

## 12) Handover Checklist

- [ ] Supabase schema created; enums defined
- [ ] Alembic baseline & migration generated
- [ ] Seed 5 organizer demo accounts (hashed passwords)
- [ ] Heroku apps created (frontend & backend); env vars set
- [ ] Procfiles committed; buildpacks configured
- [ ] Frontend store + RTK Query slices stubs
- [ ] Backend routers & services stubs
- [ ] Health checks responding

---

## 13) Next Steps

- Final review of schemas & DTOs
- Create project repos (frontend/backend) & push skeletons
- Prepare PRPs for code generation tasks
- Start implementation with minimal happy-path flows
