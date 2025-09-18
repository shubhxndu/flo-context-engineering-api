# PRD: Workshop Companion App

## 1. Overview
A companion app for workshops to help participants catch up on missed content and stay aligned with the session flow.  
- Built with Next.js (SPA) frontend and FastAPI backend.  
- Data stored in Supabase (Postgres).  

## 2. Goals
- Allow late joiners to view content they missed.  
- Provide a synchronized “current step” view.  
- Track attendance automatically.  
- Enable organizers to control and manage workshop flow.  

## 3. User Roles
### Participants
- Login with **workshop code** and **name** (email optional).  
- View current step and past modules.  
- Navigate back and forth through modules (cursor buttons).  
- Use a **“Back to Live / Current Step”** button (similar to recenter in Google Maps).  
- React with emojis.  
- Access resources section with all discussed content.  
- View agenda, elapsed time, and upcoming topics.  

### Organizers
- Create and publish sessions.  
- Pre-upload slides/modules.  
- Optionally add modules mid-session if needed.  
- Control current step (only **primary controller** can advance).  
- View participants list and total count.  

## 4. Core MVP Features
- Workshop code based login with mandatory name (email optional).  
- Automatic attendance marking at login.  
- Current step indicator and navigation controls for participants.  
- Emoji reactions per module.  
- Organizer dashboard: control session, manage content, see participant list/count.  
- Resources section with discussed content.  
- Meta info: agenda, elapsed time, upcoming topics.  

## 5. Non-Goals (Future Roadmap)
- Real-time WebSocket sync (MVP uses REST + polling).  
- Chat/Q&A.  
- Notifications.  
- Export of attendance.  
- Bookmarking/notes.  
- Scheduled workshops in advance.  
- OTP/email authentication.  
- Advanced analytics.  

## 6. Technical Architecture (MVP)
- **Frontend**: Next.js SPA.  
- **Backend**: FastAPI.  
- **Database**: Supabase (Postgres).  
- **Auth**: Workshop code + name (email optional).  
- **Sync**: REST API + periodic polling.  

## 7. Success Metrics
- Participants can log in with workshop code and access missed/current content.  
- Organizers can move session flow and see participants.  
- Navigation and “Back to Current Step” function work seamlessly.  
- Emoji reactions captured reliably.  

## 8. Risks & Assumptions
- Only one **primary controller** can advance content (avoids conflicts).  
- Polling introduces small delay but acceptable for MVP.  
- Attendance is marked once at login (no duration tracking).  
- Content can still be added mid-session if needed.  
