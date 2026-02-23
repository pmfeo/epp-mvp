# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

B2B event photo platform. Organizers create a QR code per event; guests scan it and upload photos; approved photos appear in real-time on a fullscreen slideshow on venue screens. Moderation is hybrid: AI auto-classification + mandatory human approval.

## Monorepo Structure

npm workspaces monorepo:

```
apps/backend/    NestJS RESTful API (port 3000)
apps/frontend/   React + Vite (port 5173)
packages/shared-types/   Shared TypeScript interfaces — always update here first
```

## Common Commands

```bash
# Root — run both apps in dev
npm run dev

# Backend only
npm run dev --workspace=apps/backend

# Frontend only
npm run dev --workspace=apps/frontend

# Install a backend dependency
npm install <pkg> --workspace=apps/backend

# Build all
npm run build

# Run backend tests
npm test --workspace=apps/backend

# Run a single backend test file
npm test --workspace=apps/backend -- --testPathPattern=photos.service
```

> `sharp` and `heic-convert` require native binaries. When building Docker images always target `linux/amd64` explicitly — a macOS build will fail on Linux.

## Architecture

### Backend Modules (NestJS)

| Module | Responsibility |
|---|---|
| `auth` | Supabase Auth JWT verification. A single NestJS guard validates tokens issued by Supabase using `SUPABASE_JWT_SECRET`. No custom OAuth callback — Google OAuth and email/password are handled entirely by Supabase. |
| `events` | Event CRUD, QR generation (`qrcode` → S3), cron jobs (auto-close, ZIP processing, 30-day retention) |
| `photos` | Guest upload orchestration, rate limiting, HEIC→JPEG conversion, review (approve/reject/reset) |
| `moderation` | AI provider abstraction. Mock for MVP. Real provider swapped via NestJS DI (`useClass`). |
| `slideshow` | Socket.IO gateway (`/slideshow` namespace). Manages rooms per event. |
| `storage` | **Only module that touches S3.** Never import `@aws-sdk` elsewhere. |

Global config set in `main.ts` before any module: `ValidationPipe({ whitelist: true, transform: true })`, `HttpExceptionFilter`, `enableCors`.

### Key Data Flow

**Guest upload** (`POST /events/:slug/photos`):
1. Resolve slug → eventId
2. Rate limit check (Postgres query, max 10/hour/IP/event)
3. Magic byte validation (`file-type`) → size check (≤ 10MB)
4. HEIC → JPEG (`heic-convert`) + resize to max 2000px (`sharp`)
5. Upload to S3: `events/{eventId}/photos/{photoId}.jpg`
6. Insert `photos` row with `status='pending'`
7. Fire-and-forget: `moderationService.runAI(photoId)` (errors logged, photo stays pending)

**Approve photo** → sets `approved_at`, triggers `slideshowGateway.notifyApproval()` → WS `photo-approved` broadcast.
**Reject previously-approved photo** → triggers `slideshowGateway.notifyRemoval()` → WS `photo-removed` broadcast so screen removes it immediately.

### Photo State Machine

```
upload → [pending]
  AI: verdict=fail + confidence ≥ 0.95 → [rejected, ai_auto_rejected=true]
  Organizer approve → [approved] → WS: photo-approved
  Organizer reject  → [rejected] → WS: photo-removed (only if was approved)
```

### Real-Time (Socket.IO)

Namespace: `/slideshow`. Rooms: `event:{eventId}` (supports multiple screens per event).

Client → server: `join-event { eventId }` → server replies `slideshow-init { photos: SlidePhoto[], eventConfig: { slideshowSpeed } }`.

Server → room broadcasts: `photo-approved`, `photo-removed`, `event-closed`.

On reconnect: client re-emits `join-event`, receives fresh `slideshow-init` (idempotent).

**MVP runs as single instance.** Socket.IO rooms are in-memory — does not work across multiple processes. Post-MVP: add `@socket.io/redis-adapter`.

### Storage (S3)

S3 bucket is private. Never store presigned URLs in the database — generate them fresh on every API response via `StorageService.getPresignedUrl()`. Only `s3_key` is persisted.

S3 key structure:
```
events/{eventId}/qr.png              ← public URL (QR codes are not sensitive)
events/{eventId}/photos/{photoId}.jpg ← private, presigned per request
events/{eventId}/downloads/{jobId}.zip ← private, presigned per request
```

### AI Moderation Interface

```typescript
// apps/backend/src/modules/moderation/ai-provider/ai-moderation.interface.ts
interface IAIModerationProvider {
  analyze(s3Key: string): Promise<AIModerationResult>;
}
```

To plug in a real AI service (e.g. AWS Rekognition): implement `IAIModerationProvider` and swap `useClass` in `ModerationModule`. Zero changes to `ModerationService`.

### Database (Supabase/PostgreSQL via `pg`)

Direct `pg` queries — no ORM. Always use parameterized queries (`$1, $2`).

Key constraints:
- `CREATE UNIQUE INDEX idx_events_one_active_per_org ON events(organizer_id) WHERE status = 'active'` — 1 active event per organizer enforced at DB level
- `photos.s3_key` only (no `s3_url` column)
- Slideshow query: `ORDER BY approved_at ASC, id ASC` (tiebreaker required)
- All FKs use `ON DELETE CASCADE` (events → photos → etc.)

### Cron Jobs (EventsModule, `@nestjs/schedule`)

| Frequency | Job |
|---|---|
| Every minute | Auto-close events where `closes_at <= NOW()` |
| Every hour | Process `download_jobs` with `status='pending'` (ZIP generation via `archiver`) |
| Daily | Delete events closed > 30 days ago (cascades to photos and download_jobs) |

## Frontend Routes

```
/login, /register, /auth/callback        Auth pages
/dashboard                               Organizer home (event + stats)
/dashboard/events/new                    Create event
/dashboard/events/:id                    Event detail (stats, QR, close, download)
/dashboard/events/:id/moderation         Moderation queue (tabs: Pending | Approved | Rejected)
/e/:slug                                 Guest upload page (public, mobile-first)
/screen/:eventId                         Fullscreen slideshow (public)
```

State management: Zustand. HTTP: Axios. Real-time: `socket.io-client`. Notifications: `react-hot-toast`.

## Business Rules

- Max 10 photos/hour per IP per event (TOCTOU race condition accepted for MVP; post-MVP: Redis)
- Max 10MB per photo; accepted formats: JPEG, PNG, HEIC (auto-converted), WEBP
- No guest feedback on approval/rejection — only "En revisión" after upload
- ZIP download: async job (cron), expires 30 days after event close
- JWT in localStorage (XSS risk acknowledged; post-MVP: httpOnly cookies)
- Supabase Auth handles the Google OAuth callback — no custom redirect or token-in-query-param. `supabase-js` manages token refresh automatically.

## Authentication (Supabase Auth)

Google OAuth and email/password auth are **fully delegated to Supabase Auth**. The backend never issues tokens.

**Frontend flow:**
```
Login/Register → supabase.auth.signInWithPassword() / signUp()
Google OAuth   → supabase.auth.signInWithOAuth({ provider: 'google' })
                 (Supabase handles the callback — no /auth/google/callback route needed)
Post-login     → supabase.auth.getSession() → session.access_token
               → sent as Authorization: Bearer <token> on every API request
```

**Backend guard:**
- Verifies the JWT signature with `SUPABASE_JWT_SECRET`
- Extracts `sub` (userId) and `email` from claims — same shape as a custom JWT
- Attaches `{ userId, email }` to `req.user`

**User table:**
- `auth.users` lives in Supabase's internal schema
- A `profiles` table in the public schema mirrors it via a Supabase trigger:
  ```sql
  CREATE TABLE profiles (
    id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    display_name TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
  );
  ```
- `events.organizer_id` is a FK → `profiles.id`

## Environment Variables

```bash
# apps/backend/.env
DATABASE_URL                          # Supabase PostgreSQL connection string
SUPABASE_URL                          # For any server-side Supabase client usage
SUPABASE_JWT_SECRET                   # To verify Supabase-issued JWTs in the guard
AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_REGION, S3_BUCKET_NAME
FRONTEND_URL

# apps/frontend/.env
VITE_API_URL, VITE_WS_URL
VITE_SUPABASE_URL, VITE_SUPABASE_ANON_KEY
```
