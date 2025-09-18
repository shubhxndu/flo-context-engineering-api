# PRD --- Workshop Companion App (Detailed)

**Version:** v0.8 • **Date:** 18 Sep 2025 • **Owner:** Shubhendu Mishra

---

## 1) Overview

A companion app for live workshops enabling late joiners to catch up on missed content and stay aligned with the presenter-controlled flow. - **Frontend:** Next.js (SPA) - **Backend:** FastAPI - **DB:** Supabase (Postgres) - **Sync:** REST + periodic polling (WebSockets parked)

## 2) Goals & Non-Goals

### Goals (MVP)

- Late joiner access to already-covered modules
- Synchronized **Current Module** view with organizer control
- Automatic attendance (present once at login)
- Emoji reactions (ephemeral, like Google Meet)
- Basic organizer dashboard: participant list + total count
- Meta info: agenda, elapsed time, upcoming topics
- Title always visible; optional description in submenu
- Progress bar (e.g., Module 3 of 10)
- Resources section for discussed content (accessible anytime, no expiry)
- Session end flow with "Workshop Ended --- Resources Available" page

### Non-Goals (Roadmap)

- WebSockets / real-time push
- Chat/Q&A, notifications
- Attendance exports, analytics (join time/duration)
- Bookmarks/notes, scheduled workshops
- Audit logs
- Detailed metrics
- Advanced auth (beyond demo login)

## 3) Users & Roles

### Participants

- Join via **workshop code + name** (email optional)
- Title visible always; current module displayed prominently (like PPT slide)
- Navigate past modules with left/right arrows only; progress bar shown
- View agenda, elapsed time, upcoming topics
- Resources section with links for all discussed content
- React with ephemeral emojis (vanish after display, not persistent)
- Max 500 participants per workshop

### Organizers (multiple allowed)

- **Login:** Access via **Login ID + Password** (hardcoded demo accounts: e.g., organizer1, organizer2, ...; no signup flow). Passwords stored hashed.
- **Session Creation:** After login, organizers can create a new session with title, optional description, modules.
- **Publish & Share:** Once published, an alphanumeric workshop code is generated; participants use this code to join.
- May add quick modules mid-session (edits allowed)
- Modules can be reordered before publishing.
- **Primary controller** (first organizer to press Start) exclusively advances steps.
- If primary disconnects, **any organizer can take control manually** from dashboard.
- View participant list + total count (duplicates allowed; internal IDs shown). The system will differentiate participants with the same name visually, for example, by using a different colored avatar.

## 4) User Stories (MVP)

- As a participant, I can enter a workshop code and my name to join.
- As a participant, I can see the current module and browse previously covered modules.
- As a participant, I can tap an emoji to react, and my reaction appears briefly on my UI immediately; other participants' reactions appear after the polling interval.
- As a participant, I can jump back to the current module at any time.
- As a participant, I can see workshop title always, optional description in submenu, and current module fullscreen.
- As a participant, I can track my progress via a progress bar.
- As a participant, I can access a single, combined list of all resources from all modules at the end and in the future.
- As a participant, I see a status page when the workshop ends with resources available.
- As an organizer, I can log in with ID and password to access the dashboard.
- As an organizer, I can create/publish a session and upload/reorder modules before start.
- As an organizer, I can add a quick module mid-session if needed.
- As an organizer, I can attach resources to a specific module.
- As the primary controller, I can move to next/previous module; others cannot.
- As an organizer, I can view who's in the session and the total count.
- As an organizer, I can take control if the primary disconnects by clicking a "Take Control" button on the dashboard.

## 5) Functional Requirements (High Level)

**FR-01 Auth (MVP):** Participants: workshop code + name (email optional). Organizers: Login ID + Password (hardcoded demo accounts; no signup). Passwords hashed. **FR-02 Attendance:** Mark "present" once upon successful join (no duration tracking). **FR-03 Modules:** Organizer defines ordered modules; text-based content with **fully rendered Markdown support**; medium size (fits mobile screen without scrolling). Content freezes at start but quick mid-session adds permitted. Reordering allowed before publishing. **FR-04 Step Control:** Single primary controller advances current module; first organizer to press Start is controller. If controller disconnects, others can take over manually via a "Take Control" button. **FR-05 Navigation:** Participants can navigate past modules; arrows only; **Back to Current Module** button. Progress bar displayed (e.g., 3 of 10). **FR-06 Reactions:** Ephemeral emoji reactions (👍 🎉 👏 ❤️ ✅). A user's own reaction appears immediately on their screen; reactions from others appear after the polling interval. **FR-07 Meta:** Title always visible; optional description in submenu; agenda, elapsed time, upcoming topics displayed. **FR-08 Resources:** Participants can view discussed content/resources at any time. Resources stored as text links (name, URL, type: link/pdf/video/image) in PostgreSQL. Each module can have an optional set of resources attached. At the end of the workshop, a single combined list of all resources will be available. **FR-09 Dashboard:** Participant list + total count visible to organizers (duplicates allowed). **FR-10 Sync:** Polling-based updates every 3 seconds. Pause polling if participant tab backgrounded. Emoji refresh interval: 15 seconds. **FR-11 Session End:** Organizer can end session → participants see status page with "Workshop Ended --- Resources Available".

## 6) System Flows (to be detailed)

- Default page with top menu: **Join with Code** (participant) or **Join as Organizer**
- Organizer login flow (demo accounts)
- Session creation/publish flow (organizer)
- Primary controller selection and manual transfer via "Take Control" button
- Step advance & participant sync (polling)
- Ephemeral emoji submission (15s cooldown)
- Session end flow → resources page

## 7) Data Model (Draft --- to confirm)

- **organizer_accounts**: id, login_id, password (hashed; demo accounts only)
- **workshops**: id, code (4 chars A--Z, digits, excluding 0/1/O/I), title, description, agenda(json), start_at, status(draft\|published\|live\|ended), primary_controller_id, created_by
- **modules**: id, workshop_id, index, title, content(markdown), resources(json), created_at
- **resources**: id, module_id, name, url, type(link/pdf/video/image), created_at
- **participants**: id, workshop_id, name, email?
- **organizers**: id, workshop_id, user_id, role(organizer\|controller), added_at
- **state**: workshop_id, current_module_index, elapsed_ms

## 8) API (to be detailed)

- POST /join {code, name, email?}
- GET /workshops/{code}/state (current_module, agenda, upcoming, elapsed)
- GET /workshops/{code}/modules (with resources)
- POST /workshops/{id}/step {action: next\|prev\|goto}
- POST /reactions {emoji}
- GET /organizer/{id}/participants
- POST /organizer/login {id, password}
- POST /workshops (create/publish)
- POST /modules (bulk upload, quick add)
- POST /workshops/{id}/end

## 9) Permissions & Security (MVP)

- Participants: access via code only.
- Organizers: login via demo accounts (hashed passwords). No signup flow in MVP.
- Primary controller lock on the server side.
- Basic rate limits for reactions and polling.

## 10) Telemetry & Admin (MVP)

- Minimal: counts for participants; ephemeral emoji counts; simple logs.

## 11) Risks & Mitigations

- Multi-organizer conflicts → primary controller lock with manual takeover via "Take Control" button.
- Polling delay → acceptable tradeoff for MVP.
- Mid-session edits → allow quick module add; version module index carefully.
- Demo accounts may be shared → acceptable for MVP; proper auth in roadmap.

## 12) Release Plan & Cutlines

- **Phase 1 (MVP):** All FRs above; 3s polling; ephemeral emojis.
- **Phase 2+ (Roadmap):** WebSockets, exports, analytics, notes, scheduling, audit logs, advanced metrics, proper organizer accounts.

## 13) Locked Defaults

- Resources order: append only.
- Session description: plain text.
- Resources export: parked for roadmap.
- Participants must re-enter the workshop code on every page refresh or new visit.
