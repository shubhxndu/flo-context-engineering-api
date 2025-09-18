# Technical Design Document (TDD) — Workshop Companion App

**Version:** v0.4 • **Date:** 18 Sep 2025 • **Owner:** PB&S

---

## 0) Scope

Demo implementation for a workshop companion web app (5-10 concurrent users).

- **Frontend:** Next.js **Pages Router**, SSR deployment to **Heroku**
- **Backend:** FastAPI on **Heroku**
- **DB:** Supabase Postgres
- **Sync:** REST + polling (3s; pause when tab backgrounded)
- **Auth:** Participant (code+name+session persistence), Organizer (demo login, hashed pwd)
- **State:** RTK Query for server state; cookie-based session persistence

---

## 1) Architecture Overview

- **Frontend**: Next.js SSR with Pages Router. State via RTK Query. Polling every 3s for state/modules, 15s for reaction aggregates. Session persistence via cookies.
- **Backend**: FastAPI app. Service layer with business logic, repository layer for DB access. Organizer session via cookie, participant session with cookie persistence.
- **Database**: Supabase Postgres. Tables: organizer_accounts, workshops, modules, participants, organizers, state.
- **Deployment**: Both frontend and backend hosted on Heroku (SSR for frontend). Optimized for 5-10 concurrent users (demo app).
- **Observability**: Logs, health checks. No advanced metrics in demo version.

---

## 2) Backend (FastAPI)

### Folder Structure

- backend/
  - app/
    - api/routers/
      - public.py
      - organizer.py
    - core/
      - config.py
      - security.py
      - rate-limit.py
    - services/
      - workshops.py
      - modules.py
      - participants.py
      - reactions.py
      - organizers.py
      - state.py
    - repositories/
      - workshops.py
      - modules.py
      - participants.py
      - organizers.py
      - state.py
    - models/
      - (SQLAlchemy models)
    - schemas/
      - (Pydantic request/response DTOs)
    - main.py
  - alembic/
    - versions/

### 2.2 Data Model (high-level)

- **organizer_accounts**: id, login_id, password_hash, created_at
- **workshops**: id, code, title, description, agenda_json, start_at, status, primary_controller_id, created_by, created_at
- **modules**: id, workshop_id, index, title, content_markdown, created_at
- **resources**: id, module_id, name, url, type, created_at
- **participants**: id, workshop_id, name, email_hash, created_at
- **organizers**: id, workshop_id, user_id, role, added_at
- **state**: workshop_id, current_step_index, elapsed_ms, updated_at
- **reactions**: id, workshop_id, module_id, participant_id, emoji, created_at

### 2.3 API Endpoints (skeleton)

- **Public (Participant)**

  - POST /join
  - GET /w/{code}/state
  - GET /w/{code}/modules
  - GET /w/{code}/resources
  - POST /w/{code}/react
  - GET /w/{code}/reactions/{module_id}

- **Organizer (Auth JWT)**

  - POST /organizer/login
  - POST /organizer/refresh
  - POST /organizer/logout
  - POST /o/workshops
  - POST /o/workshops/{id}/publish
  - POST /o/workshops/{id}/start
  - POST /o/workshops/{id}/end
  - POST /o/workshops/{id}/step
  - POST /o/workshops/{id}/modules
  - POST /o/workshops/{id}/modules/{module_id}/resources
  - DELETE /o/workshops/{id}/modules/{module_id}/resources/{resource_id}
  - PATCH /o/workshops/{id}/modules/reorder
  - GET /o/workshops/{id}/participants
  - GET /o/workshops/{id}/reactions
  - POST /o/workshops/{id}/takeover

### 2.4 Technical Standards

- Organizer session: JWT tokens with 1h expiry + refresh tokens (24h expiry), httpOnly, secure
- Participant session: JWT tokens with 8h expiry, cookie persistence for demo reliability
- Rate limits: reactions (3/min per user, 20/hour), joins (3/min/IP), organizer login (5/min/IP)
- Errors: standardized JSON { code, message, details? }
- Schema migrations: Alembic; demo organizers seeded
- Email Hashing: `email_hash` field stores SHA-256 hash with salt for privacy
- Security: CORS restricted, input validation via Pydantic, SQL injection prevention

### 2.5 Deployment (Heroku)

- Procfile: `web: uvicorn app.main:app --host 0.0.0.0 --port $PORT`
- Buildpacks: heroku/python
- Env vars: DB URL, COOKIE_SECRET, RATE_LIMITS, CORS_ORIGIN
- Mitigation: Use a service like Kaffeine to ping the app and prevent dyno sleep.

---

## 3) Frontend (Next.js Pages Router)

### 3.1 Folder Structure

- frontend/
  - src/
    - pages/
      - index.tsx
      - o/
        - login.tsx
        - dashboard.tsx
      - s/
        - [code].tsx
    - lib/api/
      - (RTK Query slices)
    - components/
      - JoinForm.tsx
      - OrganizerLoginForm.tsx
      - ModuleViewer.tsx
      - ProgressBar.tsx
      - NavArrows.tsx
      - BackToCurrent.tsx
      - EmojiTray.tsx
      - ResourcesList.tsx
      - EndedStatusPage.tsx
    - store/
      - index.ts
    - styles/

### 3.2 State & Networking

- RTK Query slices for participant + organizer APIs
- Polling intervals: 3s for session state/modules, 15s for reactions
- Participant identity: Cookie-based session persistence (1 day expiry) for demo reliability
- Organizer: session cookie

### 3.3 UI Standards

- Tailwind + shadcn/ui components
- Markdown rendering via react-markdown + rehype-sanitize
- Mobile-first layout; modules sized to avoid vertical scroll

### 3.4 Error Handling

- API errors surfaced via toasts; standardized error codes

### 3.5 Deployment (Heroku)

- Build: `next build` for SSR deployment
- Serve with Next.js production server
- Procfile: `web: npm start`
- Buildpacks: heroku/nodejs

---

## 4) Testing Strategy

- Backend: pytest unit + API contract tests
- Frontend: Jest/RTL for RTK Query hooks + components
- E2E smoke flow: join → publish → start → step → react → end
- Load test: simulate 10 participants + reaction bursts (demo scope)

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
  - `repositories/` (workshops, modules, participants, organizers, state)
  - `models/` (sqlalchemy models)
  - `schemas/` (pydantic DTOs)
  - `main.py`

- `alembic/` (migrations)
- `Procfile`, `requirements.txt`, `runtime.txt`

### 9.2 Data Models (Fields)

- **organizer_accounts**: id (uuid), login_id (text, unique), password_hash (text), created_at (timestamptz)
- **workshops**: id (uuid), code (char(4), unique, A–Z+digits excl. 0/1/O/I), title (text), description (text), agenda_json (jsonb), start_at (timestamptz), status (enum: draft/published/live/ended), primary_controller_id (uuid, nullable), created_by (uuid), created_at (timestamptz)
- **modules**: id (uuid), workshop_id (uuid fk), index (int), title (text), content_markdown (text), created_at (timestamptz)
- **resources**: id (uuid), module_id (uuid fk), name (text), url (text), type (enum: link/pdf/video/image), description (text, nullable), created_at (timestamptz)
- **participants**: id (uuid), workshop_id (uuid fk), name (text), email_hash (text, nullable), session_token (text, unique), created_at (timestamptz)
- **organizers**: id (uuid), workshop_id (uuid fk), user_id (uuid fk organizer_accounts), role (enum: organizer/controller), added_at (timestamptz)
- **state**: workshop_id (uuid pk/fk), current_step_index (int), elapsed_ms (bigint), updated_at (timestamptz)
- **reactions**: id (uuid), workshop_id (uuid fk), module_id (uuid fk), participant_id (uuid fk), emoji (text), created_at (timestamptz), expires_at (timestamptz)
- **organizer_sessions**: id (uuid), user_id (uuid fk), access_token_hash (text), refresh_token_hash (text), expires_at (timestamptz), created_at (timestamptz)

### 9.3 DTOs (Requests/Responses)

- **Auth**

  - OrganizerLoginRequest { loginId, password }
  - OrganizerLoginResponse { accessToken, refreshToken, expiresIn, user: { id, loginId } }
  - RefreshTokenRequest { refreshToken }
  - RefreshTokenResponse { accessToken, expiresIn }
  - JoinRequest { code, name, email? }
  - JoinResponse { sessionToken, workshop: { id, code, title, totalSteps }, state }

- **State/Modules/Resources**

  - GetStateResponse { currentStep, agenda, elapsedMs, upcoming }
  - GetModulesResponse \[ { id, index, title, contentMarkdown } ]
  - GetResourcesResponse \[ { id, moduleId, name, url, type, description } ]
  - CreateResourceRequest { name, url, type, description? }
  - CreateResourceResponse { id, name, url, type, description, createdAt }

- **Control**

  - PublishWorkshopRequest { workshopId }
  - StartWorkshopRequest { workshopId }
  - EndWorkshopRequest { workshopId }
  - StepRequest { action: next|prev|goto, index? }
  - TakeoverRequest { workshopId }
  - ParticipantsListResponse \[ { participantId, name, joinedAt } ]

- **Reactions**

  - ReactionRequest { emoji, moduleId? }
  - ReactionResponse { ok, reactionId }
  - GetReactionsResponse \[ { emoji, count, recentParticipants: \[name] } ]

### 9.4 API Standards

- JSON only; snake_case in DB, camelCase in API
- Status codes: 200/201/204; 400/401/403/404/409/429
- Error envelope: { code, message, details? } with codes: INVALID_CODE, UNAUTHORIZED, FORBIDDEN, NOT_PRIMARY, CONFLICT, RATE_LIMITED, NOT_FOUND, VALIDATION_ERROR
- Caching: ETag on state/modules responses; support If-None-Match

### 9.5 Security & Limits

- Organizer auth: JWT access tokens (1h expiry) + refresh tokens (24h expiry); httpOnly, secure
- Participant session: JWT tokens (8h expiry) with cookie persistence for demo reliability
- Token rotation: Access tokens auto-refresh; refresh tokens rotate on use
- Rate limits: reactions 3/min per user + 20/hour; joins 3/min/IP; organizer login 5/min/IP
- Input validation: All requests validated via Pydantic schemas
- Sanitization: Markdown sanitized server-side (bleach) and client-side (DOMPurify)
- Email hashing: SHA-256 with unique salt per participant
- CORS: Restricted to frontend domain only
- SQL Injection: Prevented via SQLAlchemy ORM and parameterized queries

### 9.6 Deployment (Heroku — Backend)

- Buildpack: heroku/python
- Procfile: `release: alembic upgrade head` + `web: uvicorn app.main:app --host 0.0.0.0 --port $PORT`
- Env vars: `DATABASE_URL`, `CORS_ORIGIN`, `JWT_SECRET`, `JWT_REFRESH_SECRET`, `EMAIL_SALT`, `RATE_LIMITS`, `LOG_LEVEL`
- DB: Supabase connection via `DATABASE_URL`
- Migrations: Alembic run on release phase before deployment
- Health: `/healthz` with database connectivity check
- Security: HTTPS enforced, secure headers middleware

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

    - `sessionApi.ts` (state/modules/react)
    - `organizerApi.ts` (login, workshops, control, participants)

  - `store/` (Redux store + RTK Query config)
  - `styles/` (Tailwind config and globals)
  - `utils/` (visibility, polling control, storage)
  - `config/` (env, constants, error codes)

### 10.2 Standards & Rules

- **State**: RTK Query for server state; UI state local where trivial
- **Polling**: 3s for state/modules; 15s for reaction aggregates; pause on `document.hidden`
- **Identity**: Cookie-based session persistence (1 day expiry) for demo reliability
- **Markdown**: `react-markdown` + `rehype-sanitize` (allow links, lists, headings)
- **Accessibility**: keyboard nav for arrows; ARIA for progress
- **Errors**: render standardized JSON error messages; toasts for non-blocking issues

### 10.3 Deployment (Heroku — Frontend)

- Build: `next build` for SSR deployment
- Serve: Next.js production server
- Procfile: `web: npm start`
- Buildpacks: heroku/nodejs

---

## 11) Non-Functional Requirements

- **Performance**: initial load < 2s on mid-tier 4G; module view interactions < 100ms
- **Reliability**: graceful polling resume after network loss
- **Scalability**: up to 10 concurrent participants per workshop (demo scope)
- **Security**: basic OWASP A1–A3 hygiene; sanitized markdown; cookie protections; email hashing.

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
