# PRD — Event Photo Platform (EPP) MVP

**Status:** Pre-development
**Last updated:** 2026-02-22

---

## 1. Product Overview

### Problem

At live events (weddings, corporate dinners, brand activations), guests capture dozens of photos on their phones that never reach the venue screens or the organizer's archive. Existing solutions either require app installs, lack real-time display, or have no moderation layer — making them unsuitable for professional B2B events.

### Solution

A B2B SaaS platform where event organizers generate a QR code per event. Guests scan it, upload photos from a mobile browser, and approved photos appear in real-time on a fullscreen slideshow on venue screens — with no app install required on either side.

### Value Proposition

| Stakeholder | Value |
|---|---|
| Organizer | Curated, branded live slideshow with full moderation control |
| Guest | Zero-friction photo sharing — scan, upload, done |
| Venue screen | Plug-and-play fullscreen display, auto-reconnects |

---

## 2. Actors

| Actor | Description | Auth |
|---|---|---|
| **Organizer** | B2B customer who creates and manages events | Email/password or Google OAuth |
| **Guest** | Event attendee who uploads photos | None (anonymous, public URL) |
| **Screen** | Venue display device running the slideshow | None (public URL by eventId) |

---

## 3. MVP Scope

### In Scope

- Organizer authentication (email/password + Google OAuth)
- Event creation with QR code generation
- One active event per organizer at a time
- Guest photo upload (JPEG, PNG, WEBP, HEIC) via public mobile page
- AI-assisted moderation (mock for MVP) + mandatory human approval
- Real-time slideshow via Socket.IO (fullscreen, venue screens)
- Moderation queue UI (Pending / Approved / Rejected tabs)
- Async ZIP download of approved photos (cron-based)
- Auto-close events at scheduled time
- 30-day data retention after event close

### Explicitly Out of Scope (MVP)

See Section 10 for the complete post-MVP list.

---

## 4. Functional Requirements

### Organizer (FR-O)

| ID | Requirement |
|---|---|
| FR-O01 | Register with email/password |
| FR-O02 | Log in with email/password or Google OAuth |
| FR-O03 | Create an event (name, slug, closes_at, slideshow speed) |
| FR-O04 | Only one active event at a time; creating a second requires closing the first |
| FR-O05 | View event detail: stats (total / pending / approved / rejected), QR code, status |
| FR-O06 | Share or download the QR code PNG |
| FR-O07 | Manually close an active event |
| FR-O08 | View moderation queue with three tabs: Pending, Approved, Rejected |
| FR-O09 | Approve a pending photo (triggers real-time slideshow update) |
| FR-O10 | Reject a photo (triggers real-time removal if previously approved) |
| FR-O11 | Reset a rejected photo back to pending |
| FR-O12 | Request ZIP download of all approved photos (async, receive link when ready) |
| FR-O13 | Events auto-close when `closes_at` is reached |
| FR-O14 | Closed events and all associated data are deleted after 30 days |

### Guest (FR-G)

| ID | Requirement |
|---|---|
| FR-G01 | Access the upload page by scanning a QR code (no login required) |
| FR-G02 | Upload one or more photos (JPEG, PNG, WEBP, HEIC) |
| FR-G03 | Receive confirmation that the photo is "En revisión" after upload |
| FR-G04 | Receive no feedback about approval or rejection outcome |
| FR-G05 | Be rate-limited to 10 uploads/hour per IP per event |
| FR-G06 | Upload page is mobile-first and works on any modern mobile browser |
| FR-G07 | Upload is blocked if the event is closed or not found |

### Screen / Slideshow (FR-S)

| ID | Requirement |
|---|---|
| FR-S01 | Display a fullscreen slideshow at `/screen/:eventId` (no login required) |
| FR-S02 | Receive all currently approved photos on join (via `slideshow-init`) |
| FR-S03 | Add new photos to the slideshow in real-time (`photo-approved` event) |
| FR-S04 | Remove photos from the slideshow in real-time when rejected (`photo-removed` event) |
| FR-S05 | Display a "event closed" state when the event ends (`event-closed` event) |
| FR-S06 | Reconnect automatically and re-sync state on network interruption |
| FR-S07 | Support multiple screens connected simultaneously to the same event |

---

## 5. User Flows

### 5.1 Guest Upload Flow

```
Guest scans QR code
    │
    ▼
GET /e/:slug  ──► event not found or closed? → show error page
    │
    ▼
Guest selects photo(s) on upload page
    │
    ▼
POST /events/:slug/photos
    ├─ Rate limit exceeded (10/hr/IP/event)? → 429 Too Many Requests
    ├─ Invalid file type or > 10MB? → 400 Bad Request
    └─ OK
        ├─ HEIC converted to JPEG, resized to max 2000px
        ├─ Uploaded to S3
        ├─ DB row inserted: status = 'pending'
        └─ AI moderation triggered (fire-and-forget)
            │
            ▼
Guest sees "En revisión" — no further feedback
```

### 5.2 Moderation Flow

```
Organizer opens /dashboard/events/:id/moderation
    │
    ▼
[Pending tab] — photos awaiting review
    ├─ AI auto-rejected (confidence ≥ 0.95)? → shown in Rejected tab (ai_auto_rejected=true)
    ├─ Organizer clicks Approve
    │       └─ status → 'approved', approved_at set
    │           └─ WS broadcast: photo-approved → screens add photo
    └─ Organizer clicks Reject
            └─ status → 'rejected'
                └─ WS broadcast: photo-removed (only if was previously approved)

[Approved tab] — organizer can reject approved photos
[Rejected tab] — organizer can reset to pending
```

### 5.3 Slideshow Display Flow

```
Screen opens /screen/:eventId
    │
    ▼
Socket.IO connects to /slideshow namespace
    │
    ▼
Client emits: join-event { eventId }
    │
    ▼
Server replies: slideshow-init { photos: SlidePhoto[], eventConfig: { slideshowSpeed } }
    │
    ▼
Screen renders slideshow loop
    │
    ├─ WS: photo-approved  → append photo to slideshow
    ├─ WS: photo-removed   → remove photo from slideshow immediately
    └─ WS: event-closed    → show "event ended" state

On disconnect/reconnect → client re-emits join-event → receives fresh slideshow-init
```

---

## 6. Data Model

### Tables

#### `profiles`

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | FK → `auth.users(id)` ON DELETE CASCADE (Supabase internal) |
| `display_name` | TEXT | Optional display name |
| `created_at` | TIMESTAMPTZ | |

Populated automatically via a Supabase trigger on `auth.users` insert. `events.organizer_id` references `profiles.id`.

#### `events`

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `organizer_id` | UUID FK → profiles | ON DELETE CASCADE |
| `name` | TEXT | Display name |
| `slug` | TEXT UNIQUE | Used in guest upload URL |
| `status` | TEXT | `'active'` \| `'closed'` |
| `closes_at` | TIMESTAMPTZ | Auto-close trigger |
| `slideshow_speed` | INTEGER | Milliseconds per slide |
| `created_at` | TIMESTAMPTZ | |

**Key constraint:** `UNIQUE INDEX ON events(organizer_id) WHERE status = 'active'` — enforces one active event per organizer at DB level.

#### `photos`

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `event_id` | UUID FK → events | ON DELETE CASCADE |
| `s3_key` | TEXT | Path in S3; never store presigned URL |
| `status` | TEXT | `'pending'` \| `'approved'` \| `'rejected'` |
| `ai_auto_rejected` | BOOLEAN | True if AI rejected with confidence ≥ 0.95 |
| `approved_at` | TIMESTAMPTZ | Set on approval; used for slideshow ordering |
| `created_at` | TIMESTAMPTZ | |

**Slideshow ordering:** `ORDER BY approved_at ASC, id ASC` (tiebreaker required).

#### `download_jobs`

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `event_id` | UUID FK → events | ON DELETE CASCADE |
| `status` | TEXT | `'pending'` \| `'processing'` \| `'done'` \| `'failed'` |
| `s3_key` | TEXT | Set when ZIP is ready |
| `created_at` | TIMESTAMPTZ | |
| `completed_at` | TIMESTAMPTZ | |

### Photo State Machine

```
                    ┌─────────────────────────────┐
                    │                             │
upload ──────► [pending]                         │
                    │                             │
                    ├─ AI fail + conf ≥ 0.95 ──► [rejected] ◄─ Organizer reject
                    │   (ai_auto_rejected=true)       │
                    │                             reset │
                    └─ Organizer approve ──► [approved] ─────────────────┘
                                                  │
                                              WS: photo-approved
                                      (photo-removed on reject from approved)
```

---

## 7. API Surface

### Auth Module

> **Auth is handled by Supabase Auth — the backend exposes no auth endpoints.**
>
> Registration, login (email/password), and Google OAuth are performed entirely on the frontend via `@supabase/supabase-js`. Supabase issues and refreshes JWTs automatically. The NestJS backend only verifies the JWT on protected routes using `SUPABASE_JWT_SECRET`.

| Client action | Mechanism |
|---|---|
| Email/password register | `supabase.auth.signUp({ email, password })` |
| Email/password login | `supabase.auth.signInWithPassword({ email, password })` |
| Google OAuth | `supabase.auth.signInWithOAuth({ provider: 'google' })` |
| Token refresh | Automatic via `supabase-js` (no backend involvement) |
| Authenticated API call | `Authorization: Bearer <supabase_access_token>` header |

### Events Module

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/events` | JWT | List organizer's events |
| POST | `/events` | JWT | Create event + generate QR |
| GET | `/events/:id` | JWT | Event detail + stats |
| PATCH | `/events/:id/close` | JWT | Manually close event |
| POST | `/events/:id/download` | JWT | Request ZIP download job |
| GET | `/events/:id/download/:jobId` | JWT | Poll job status / get presigned URL |

### Photos Module

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/events/:slug/photos` | None | Guest upload (multipart/form-data) |
| GET | `/events/:id/photos` | JWT | List photos (filterable by status) |
| PATCH | `/events/:id/photos/:photoId/approve` | JWT | Approve photo |
| PATCH | `/events/:id/photos/:photoId/reject` | JWT | Reject photo |
| PATCH | `/events/:id/photos/:photoId/reset` | JWT | Reset rejected → pending |

### Slideshow / Socket.IO

**Namespace:** `/slideshow`
**Rooms:** `event:{eventId}`

| Direction | Event | Payload | Description |
|---|---|---|---|
| Client → Server | `join-event` | `{ eventId: string }` | Join slideshow room |
| Server → Client | `slideshow-init` | `{ photos: SlidePhoto[], eventConfig: { slideshowSpeed } }` | Initial state on join |
| Server → Room | `photo-approved` | `{ photo: SlidePhoto }` | New photo to display |
| Server → Room | `photo-removed` | `{ photoId: string }` | Remove photo immediately |
| Server → Room | `event-closed` | `{}` | Event has ended |

---

## 8. Non-Functional Requirements

### Idioma de la Interfaz

Todas las comunicaciones visibles al usuario en la interfaz deben estar en **español**. Esto incluye sin excepción:

- Mensajes de error (validación de formularios, errores de API, errores de red)
- Notificaciones y toasts (éxito, advertencia, error)
- Estados de carga y pantallas vacías
- Textos de confirmación y modales
- Mensajes del flujo de subida (guest upload), moderación y slideshow
- Cualquier otro texto que el usuario pueda leer en la aplicación



### Upload Limits

| Parameter | Limit |
|---|---|
| Max file size | 10 MB |
| Accepted formats | JPEG, PNG, WEBP, HEIC (auto-converted to JPEG) |
| Max image output dimension | 2000px (longest side, via `sharp`) |
| Rate limit | 10 uploads / hour / IP / event |

### Performance

- Presigned S3 URLs generated fresh on every API response — never cached in DB.
- Slideshow ordering is stable: `approved_at ASC, id ASC`.
- AI moderation is fire-and-forget; failures are logged and do not block the upload response.

### Security Notes (MVP trade-offs acknowledged)

| Area | MVP Approach | Post-MVP Target |
|---|---|---|
| JWT storage | localStorage (XSS risk) | httpOnly cookies |
| Google OAuth | Supabase Auth handles callback (no token in query param) | Same |
| Rate limiting | Postgres query (TOCTOU race condition) | Redis atomic counter |
| S3 access | Private bucket, presigned URLs per request | Same |
| File validation | Magic byte check (`file-type`) + size check | Same |

### Availability

- MVP runs as a **single instance**. Socket.IO rooms are in-memory.
- Cron jobs run within the single NestJS process (`@nestjs/schedule`).

---

## 9. Technical Architecture

### Monorepo Layout

```
apps/backend/         NestJS RESTful API — port 3000
apps/frontend/        React + Vite — port 5173
packages/shared-types/  Shared TypeScript interfaces (update here first)
```

### Backend Module Responsibilities

| Module | Responsibility |
|---|---|
| `auth` | Supabase Auth JWT verification. One NestJS guard validates `SUPABASE_JWT_SECRET`; extracts `userId` + `email` from claims. No custom OAuth routes. |
| `events` | Event CRUD, QR generation (`qrcode` → S3), cron jobs |
| `photos` | Guest upload orchestration, rate limiting, HEIC→JPEG, review actions |
| `moderation` | AI provider abstraction. Mock for MVP. Swap via NestJS DI `useClass`. |
| `slideshow` | Socket.IO gateway (`/slideshow` namespace). Per-event rooms. |
| `storage` | **Sole S3 interface.** No other module imports `@aws-sdk`. |

**Global NestJS config** (set in `main.ts`): `ValidationPipe({ whitelist: true, transform: true })`, `HttpExceptionFilter`, `enableCors`.

### Storage Rules

- S3 bucket is **private**.
- Only `s3_key` is persisted in the database. Presigned URLs are generated on demand via `StorageService.getPresignedUrl()`.
- S3 key structure:

```
events/{eventId}/qr.png                    ← public URL (QR codes)
events/{eventId}/photos/{photoId}.jpg      ← private, presigned per request
events/{eventId}/downloads/{jobId}.zip     ← private, presigned per request
```

### AI Moderation Interface

Implement `IAIModerationProvider` to swap in a real AI service (e.g. AWS Rekognition) with zero changes to `ModerationService`:

```typescript
interface IAIModerationProvider {
  analyze(s3Key: string): Promise<AIModerationResult>;
}
```

### Cron Jobs

| Frequency | Job |
|---|---|
| Every minute | Auto-close events where `closes_at <= NOW()` |
| Every hour | Process `download_jobs` with `status='pending'` (ZIP via `archiver`) |
| Daily | Delete events closed > 30 days ago (cascades to photos and download_jobs) |

### Database

- **Provider:** Supabase (PostgreSQL)
- **Driver:** `pg` — direct parameterized queries (`$1, $2`). No ORM.
- All foreign keys use `ON DELETE CASCADE`.

### Frontend Stack

| Concern | Library |
|---|---|
| Framework | React + Vite |
| State | Zustand |
| HTTP | Axios |
| Real-time | `socket.io-client` |
| Notifications | `react-hot-toast` |

### Frontend Routes

| Route | Description | Auth |
|---|---|---|
| `/login`, `/register` | Auth pages | Public |
| `/auth/callback` | Supabase OAuth redirect target — `supabase-js` reads session from URL hash automatically | Public |
| `/dashboard` | Organizer home — event list + stats | Private |
| `/dashboard/events/new` | Create event | Private |
| `/dashboard/events/:id` | Event detail (stats, QR, close, download) | Private |
| `/dashboard/events/:id/moderation` | Moderation queue | Private |
| `/e/:slug` | Guest upload page (mobile-first) | Public |
| `/screen/:eventId` | Fullscreen slideshow | Public |

---

## 10. Out of Scope (Post-MVP)

| Feature | Notes |
|---|---|
| Multi-instance / horizontal scaling | Requires `@socket.io/redis-adapter` |
| Redis rate limiting | Replaces Postgres-based counter; eliminates TOCTOU |
| httpOnly cookie JWT | Eliminates XSS risk from localStorage |
| Real AI moderation provider | Swap `MockAIModerationProvider` for AWS Rekognition or similar |
| Guest approval/rejection notifications | Currently: no feedback beyond "En revisión" |
| Multiple active events per organizer | DB constraint currently enforces one |
| Payment / subscription billing | Not in current architecture |
| Analytics / success metrics | Not in current architecture |
| Native mobile app | Guest flow is mobile web only |
| Video upload | Photos only |
| Custom branding per event | Not in current architecture |
