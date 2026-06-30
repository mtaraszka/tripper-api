# Tripper API — Architecture & Specification

**Project:** Tripper API (backend repository: `tripper-api`)
**Document version:** v1.0 (consolidated from specification v6.1, architecture v1.0, and backend architecture v1.1)
**Status:** conceptual phase, pre-implementation

This document consolidates all backend-relevant decisions for Tripper: business logic, data model, technology stack, API contract, and infrastructure. It intentionally excludes frontend-only concerns (UI screens, design system, Angular component architecture) — those are documented separately in the `tripper-frontend` repository.

---

## 1. Project Goal

Tripper is a web application (SPA) supporting individual and group travelers throughout every stage of a trip. It combines four core areas:

- **Interactive Planning (Timeline & Map):** Building a chronological itinerary alongside route visualization and geographic calculations.
- **360° Finance & Budget Management:** Breaking down a global budget into daily limits, with two-layer cost tracking (estimated vs. actual), supporting multiple currencies.
- **Live Trip Assistant:** A dynamic in-trip dashboard for checking off completed activities and flexibly reconfiguring the plan when plans change.
- **Memories & Digital Album:** Collecting photos and videos in the context of specific days and itinerary points, intended for later album/photobook generation.
- **Group Collaboration:** Sharing a trip's structure between participants while preserving full privacy and financial autonomy for each one.

---

## 2. Trip Lifecycle (Trip State Management)

The backend logic and data exposed by the API depend heavily on the trip's logical status.

### 2.1 States and Their Meaning

| Status | Business Context | System Behavior |
|---|---|---|
| `draft` | Trip still being planned — incomplete, in progress. | Full editing of activities, drag & drop reordering, route recalculation, cost estimation, daily budget allocation. Album module locked. |
| `upcoming` | Trip ready — plan finalized, awaiting start. | Same as above. The user can revert to `draft` at any time to keep editing before departure. |
| `active` | Live mode (trip in progress). | The timeline becomes an interactive checklist. Users can check off visited places and enter actual expenses in real time. Free-form plan modification remains possible without a state change. Active budget alerts. |
| `archived` | Historical trip (completed). | Logistics and schedule become locked for editing. The timeline becomes a "Storyline" (memory stream). Full access to media upload, statistics, and photobook generation. |

### 2.2 State Transitions: Trip Start and End

Design decision: **the user controls state transitions**, not a rigid date-based schedule — e.g., a user might want to stay on a trip one extra day, and the system should not block or force a state change against their will.

**Mechanism:** a toggle in trip settings — *"Automatically start/end the trip according to planned dates."*

- **Enabled** → the system automatically switches `upcoming → active` on `startDate` and `active → archived` on `endDate` (or the day after).
- **Disabled** (flexible approach) → the user manually marks the start and end via a UI action.

> **To consider during implementation:** a state transition log/history (e.g., a list of timestamps for each status change) — useful for future statistics like "Your trip lasted 1 day longer than planned" in Tripper Wrapped.

---

## 3. Multi-Tenancy and Data Isolation (Multi-User Architecture)

The system supports group travel with a hybrid data structure: the trip's main skeleton is shared, while the intent and financial layer is strictly personalized.

- **Shared Itinerary:** The trip creator (Owner) generates an access code. Invited participants pull the plan and see the same days, activities, times, and map routes. Only the Owner can edit the global structure.
- **Autonomous Budgets:** Each group member defines their own private budget limit and allocates portions of it to individual days. Actual amounts spent, as entered by a participant, are invisible to others.
- **Opt-Out Mechanism (Participation Selector):** If a group member doesn't want to take part in a given activity (e.g., an expensive restaurant), they flag non-participation. That point becomes visually grayed out **on their own** timeline, fully excluded from their individual daily budget calculations, and omitted from their final statistics.
  > **Clarification:** an activity marked as skipped by one participant **remains fully visible** on the shared timeline and map for the rest of the group. The opt-out flag is purely personal and never removes or hides the activity from the shared trip structure.

---

## 4. Data Model (TypeScript)

### 4.1 User

**Design assumptions:**
- Login: **email + password** and **Google OAuth**.
- Avatar/profile photo: available **from MVP**, not deferred to the social version.
- `username`: **unique and required from the start**, deliberately provided by the user (not auto-generated) — to avoid a costly migration (`UNIQUE NOT NULL` on a table with existing data) later, when username will be needed for public profiles (v3.0).
- `id`: **UUID, not auto-increment** — because a user identifier may appear in public URLs/API responses in the future (profiles in v3.0); a sequential ID would reveal the number of accounts in the system and make them easier to enumerate.
- **Date of birth deliberately omitted** from the current model — see rationale in section 4.1.3.

```typescript
export type AuthProvider = 'local' | 'google';

export interface User {
  id: string;                   // UUID
  email: string;                 // unique
  username: string;              // unique, required
  passwordHash?: string;         // optional — absent for Google-only accounts
  authProvider: AuthProvider;
  displayName: string;           // name shown in the UI (e.g. on a shared group timeline)
  avatarUrl?: string;
  createdAt: Date;
}
```

#### 4.1.1 Registration Flow

**Email + password:**
A single form — email, password, username — filled out once. Username/email uniqueness is ideally validated live (debounce + API call). The account is created complete from the start.

**Google OAuth:**
1. The user logs in via Google → the backend receives email, name, avatar.
2. If the email already exists in the database → standard login.
3. If this is a new user → the data is held temporarily in a session/token, **without writing to the `users` table**.
4. The user goes through a short "Complete profile" step — providing a unique `username` (it may be pre-suggested based on their name, but must be consciously confirmed/changed, no auto-submit).
5. Only once a valid, unique `username` is provided is the `User` record **created** in the database.

> Deliberately: a `User` record never exists in the database without a `username` — this avoids working around the `UNIQUE NOT NULL` constraint with placeholders and the future migrations that would entail.

**`username` validation:** suggested rules — 3–20 characters, `a-z0-9_`, no spaces. Side benefit: in v3.0 it can serve as a readable identifier in the URL (`/profile/:username`) instead of a UUID.

#### 4.1.2 Open Topic: Multiple Login Methods on One Account

Currently one account = one login method (`authProvider` as a single value). Linking accounts (e.g. email+password and Google on the same account) has not yet been designed — to be considered if the need arises.

#### 4.1.3 Architecture Note: Date of Birth (deferred to v3.0)

Deliberately **omitted from the current model**. The main identified use case is **age verification** for minor-related restrictions in the social layer (v3.0) — and this requires the field to be **required**, not optional (optional age verification verifies nothing, since a minor could simply leave it blank).

**To do before implementing social features (v3.0):** implement `dateOfBirth` as a **required** field, collected in a dedicated onboarding step (similar to `username` during Google OAuth) — not as an empty/optional field added "just in case" earlier. Private by default, not shown publicly unless the user explicitly consents to disclosure (requires a separate privacy-settings mechanism at the profile-field level, since `dateOfBirth` likely won't be the only such field in the future).

### 4.2 Trip and Its Elements

```typescript
export type ActivityType = 'accommodation' | 'sightseeing' | 'food' | 'transport';
export type TransportType = 'car' | 'walk' | 'public_transport' | 'bicycle';
export type TripStatus = 'draft' | 'upcoming' | 'active' | 'archived';
export type TripType = 'domestic' | 'international' | 'mixed';

export interface Location {
  address: string;
  lat: number;
  lng: number;
}

export interface MediaAsset {
  id: string;
  url: string;
  thumbnailUrl?: string; // for gallery grid rendering performance
  type: 'image' | 'video';
  uploadedAt: Date;
  caption?: string;
  activityId?: string; // contextual link to an activity
}

export interface Activity {
  id: string;
  title: string;
  startTime: string; // Format "HH:mm"
  durationMinutes: number;
  type: ActivityType;
  location: Location;
  distanceFromPreviousKm?: number; // calculated via Maps API
  transportTypeToNext?: TransportType;
  notes?: string;
}

// Personal state of an activity for a specific user within a group
export interface UserActivityState {
  activityId: string;
  isVisited: boolean;       // checked off in Live Checklist mode
  actualCost: number;        // actual cost incurred by the user
  actualCostCurrency: string; // ISO 4217 — currency of the actual cost
  isAttending: boolean;       // Opt-Out mechanism flag
}

export interface DayBudget {
  date: Date;
  plannedLimit: number;  // financial limit for the given day
  currency: string;       // ISO 4217
}

export interface UserTripPreferences {
  userId: string;
  totalLimit: number; // total user budget for the trip
  currency: string;    // default/primary preference currency
  dayBudgets: DayBudget[]; // budget split across days
  activitiesStates: UserActivityState[]; // personal costs and statuses for activities
}

export interface TripDay {
  id: string;
  date: Date;
  activities: Activity[];
  photos: MediaAsset[]; // general photos for the day
}

export interface Trip {
  id: string;
  ownerId: string; // ID of the trip creator
  sharedWithUserIds: string[]; // list of group participants
  title: string;
  destination: string;
  startDate: Date;
  endDate: Date;
  status: TripStatus;
  tripType: TripType;
  availableCurrencies: string[]; // ISO 4217, e.g. ['PLN'], ['EUR', 'CZK']
  autoStartEnd: boolean; // whether the state switches automatically by date, or manually
  days: TripDay[];
  currentUserPreferences?: UserTripPreferences; // data injected for the logged-in user
}
```

### 4.3 Multi-Currency — General Rules

- Trip type (`tripType`) determines the UI in the trip creator: for `domestic` a single-select currency field appears, for `international`/`mixed` — multi-select.
- Both the daily budget (`DayBudget`) and the actual cost of a single activity (`UserActivityState.actualCost`) have their own, independent currency field — because different days/activities within the same trip may be settled in different currencies.
- **Currency exchange conversion: not in the first version.** Amounts are shown "as-is" (raw values, no conversion to a reference currency). Automatic conversion (e.g. via the NBP API or exchangerate-api) is deliberately deferred until there's a real need for it in summaries.

### 4.4 Decisions Future-Proofing Later Versions

The following model extensions are **not required by the MVP**, but have been included in the database schema from Phase 0. The cost of omitting them is a migration on a table with production data — far more expensive than adding a column with a default value at table-creation time.

Detailed rationale and TypeORM code examples are in section 7.2 (Schema Design Decisions) below.

| Column | Table | Default value | Active from |
|---|---|---|---|
| `visibility` | `trips` | `'private'` | v3.0 — social feed, public trips |
| `deleted_at` | `users`, `trips`, `activities` | `null` | v2.0/v3.0 — soft delete, GDPR |
| `role` | `trip_members` | `'member'` | v1.x Phase 7 — flexible collaboration permissions |
| `estimated_cost` + `estimated_cost_currency` | `activities` | `null` | v1.x Phase 5 — budget calculations |
| `auth_providers` (JSONB) | `users` | `["local"]` / `["google"]` | once account linking is implemented |

---

## 5. Technology Stack

### 5.1 Adopted Priorities

Priorities established before choosing technologies (order matters when resolving disputed decisions):

1. Long-term scalability
2. Developer Experience (DX)
3. Ease of maintenance
4. Speed to MVP

### 5.2 Stack Overview

| Layer | Technology | Version (approx.) |
|---|---|---|
| Backend | NestJS (Node.js + TypeScript) | latest stable |
| Real-time | Socket.io (via NestJS WebSocket Gateway) | — |
| Database | PostgreSQL | 16+ |
| ORM | TypeORM | — |
| Cache / Real-time adapter | Redis | — |
| Object Storage | S3-compatible (MinIO locally, Cloudflare R2 in production) | — |
| External API — Maps | Google Maps Platform (Places, Directions, Geocoding) | — |
| External API — Auth | Google OAuth 2.0 (via Passport.js) | — |

### 5.3 NestJS Instead of Plain Express

**A key decision for someone experienced with Angular, learning Node.js.**

NestJS is architecturally a near-twin of Angular: `@Injectable()`, `@Module()`, `@Guard()`, `@Interceptor()` — the same concepts and decorator syntax. Instead of learning two mental models at once (Angular + a custom Express structure), a single one is used throughout.

Plain Express would mean designing everything from scratch: project structure, error handling, validation, middleware, DI — everything NestJS provides and enforces out of the box. At this project's scale, that's wasted effort with no payoff.

### 5.4 Database: PostgreSQL + TypeORM

**PostgreSQL instead of MongoDB** — Tripper's data model is strongly relational. The hierarchy `User → Trip → TripDay → Activity → UserActivityState` has strict foreign keys. Relational guarantees (transactions, FK constraints) are critical for budget and trip-state consistency.

Flexibility where needed: `JSONB` columns for user preferences and `availableCurrencies` (`Trip`).

**TypeORM** — NestJS's native ORM:
- Migrations (the schema will evolve along the roadmap in section 9).
- Full TypeScript support and UUIDs as primary keys (per the decision in section 4.1).
- The `@Entity()` decorator stays consistent with the rest of the NestJS style.

### 5.5 Real-time (WebSocket): Socket.io via NestJS WebSocket Gateway

Requirement: changes made by one group participant must be visible to others instantly, without a refresh (collaboration like Google Docs).

SSE (Server-Sent Events) is ruled out — it's one-directional (server → client) and doesn't support sending client-side changes back through the same channel.

**Socket.io** via NestJS's `@WebSocketGateway()`:
- Native NestJS integration (decorators, guards, DI work the same as in HTTP).
- `rooms` map perfectly onto `trip/:id` — each participant joins their trip's room and receives only that trip's updates.
- The Redis adapter (`@socket.io/redis-adapter`) allows scaling to multiple server instances without changing application logic.

**When it's built:** Real-time is only required starting in **v1.x Phase 7 (group collaboration)**. The architecture must support it from the start, though — hence Redis and the Gateway are in the stack from day one, with the Gateway left empty until Phase 7.

### 5.6 Redis

Three use cases:
1. **Socket.io adapter** — WebSocket synchronization across server instances.
2. **Cache** — Google Maps query results (routes, distances) are expensive and change slowly; a TTL cache eliminates repeated identical queries during timeline drag & drop.
3. **Rate limiting** — protection against exceeding Google Maps API limits.

### 5.7 Object Storage: MinIO (dev) → Cloudflare R2 (production)

Photos and videos (Memories module, v2.0) don't go into PostgreSQL or the server's disk.

**Vendor lock-in-free strategy:** the `@aws-sdk/client-s3` library supports any S3-compatible API by swapping the `endpoint`. Application code doesn't change between environments:

```typescript
// Local dev — MinIO
endpoint: 'http://localhost:9000'

// Production — Cloudflare R2
endpoint: 'https://<account>.r2.cloudflarestorage.com'
```

MinIO runs locally via Docker Compose (see section 10). R2 was chosen for production due to zero egress fees (outbound data transfer) — important when serving photos directly to the browser.

### 5.8 Hosting (production / portfolio environment)

Requirement: free, private portfolio project.

| Layer | Service | Notes |
|---|---|---|
| Backend (NestJS) | **Render** (free tier) | Auto-deploy from GitHub; sleeps after 15 min of inactivity |
| PostgreSQL | **Neon** | PostgreSQL 16, 0.5 GB, connection pooling; doesn't sleep |
| Redis | **Upstash** | Serverless Redis, 10,000 req/day, 256 MB; sufficient for portfolio use |
| Object Storage | **Cloudflare R2** | 10 GB/mo, 1M req/mo, zero egress fee |
| Google OAuth | **Google Cloud** | Free |
| Google Maps | **Google Maps Platform** | $200 credit/mo — fully free at low traffic; a card is required to register but won't be charged |

**Note on Render:** cold start after sleeping takes about 30–50 seconds. Acceptable for a portfolio project. Workaround: a free cron ping every 10 minutes (e.g. cron-job.org). No-sleep alternative: **Fly.io** (3 shared-CPU machines in the free tier, more technical CLI-based setup).

---

## 6. Project Structure

```
tripper-api/
  src/
    auth/             → registration, JWT, Google OAuth, refresh tokens
    users/            → user profile, preferences (theme, language)
    trips/            → trip CRUD, state management, access code
    activities/       → activities, drag & drop ordering, opt-out mechanism
    budget/           → daily limits, actual costs, multi-currency
    realtime/         → WebSocket Gateway (Socket.io, active from v1.x)
    media/            → photo/video upload, S3 integration
    maps/             → Google Maps API proxy
    common/           → exception filters, interceptors, decorators, pipes
    config/           → environment configuration (ConfigModule)
    database/         → TypeORM configuration, migrations
  test/               → e2e tests
  docker-compose.yml
  .env.example
```

### 6.1 Structure Within a Module (consistent across all modules)

```
trips/
  trips.module.ts
  trips.controller.ts       ← HTTP handling, validation decorators
  trips.service.ts          ← business logic
  trips.repository.ts       ← database queries (TypeORM)
  dto/
    create-trip.dto.ts      ← input validation (class-validator)
    update-trip.dto.ts
    trip-response.dto.ts    ← response shape (class-transformer)
  entities/
    trip.entity.ts          ← TypeORM entity
    trip-day.entity.ts
  guards/
    trip-owner.guard.ts     ← trip ownership check
```

### 6.2 Logic Organization Principle

- **Controller** — HTTP handling only: routing, DTO validation, calling the service, returning the response. Zero business logic.
- **Service** — domain logic: state transition rules, budget calculations, orchestration across repositories.
- **Repository** — TypeORM queries only. The service never writes queries directly — always through the repository.

---

## 7. Database Schema

### 7.1 ERD

```mermaid
erDiagram
  users {
    uuid id PK
    varchar email UK
    varchar username UK
    varchar password_hash
    jsonb auth_providers
    varchar display_name
    varchar avatar_url
    timestamp created_at
    timestamp deleted_at
  }

  trips {
    uuid id PK
    uuid owner_id FK
    varchar title
    varchar destination
    date start_date
    date end_date
    varchar status
    varchar trip_type
    varchar visibility
    jsonb available_currencies
    boolean auto_start_end
    varchar invite_code UK
    timestamp created_at
    timestamp updated_at
    timestamp deleted_at
  }

  trip_members {
    uuid trip_id FK
    uuid user_id FK
    varchar role
    timestamp joined_at
  }

  trip_days {
    uuid id PK
    uuid trip_id FK
    date date
    int sort_order
  }

  activities {
    uuid id PK
    uuid trip_day_id FK
    varchar title
    time start_time
    int duration_minutes
    varchar type
    varchar location_address
    decimal location_lat
    decimal location_lng
    decimal estimated_cost
    varchar estimated_cost_currency
    float distance_from_prev_km
    varchar transport_type_to_next
    text notes
    int sort_order
    timestamp deleted_at
  }

  user_trip_preferences {
    uuid id PK
    uuid trip_id FK
    uuid user_id FK
    decimal total_limit
    varchar currency
  }

  day_budgets {
    uuid id PK
    uuid user_trip_pref_id FK
    uuid trip_day_id FK
    decimal planned_limit
    varchar currency
  }

  user_activity_states {
    uuid id PK
    uuid user_trip_pref_id FK
    uuid activity_id FK
    boolean is_visited
    boolean is_attending
    decimal actual_cost
    varchar actual_cost_currency
  }

  media_assets {
    uuid id PK
    uuid trip_day_id FK
    uuid activity_id FK
    uuid uploaded_by FK
    varchar url
    varchar thumbnail_url
    varchar type
    text caption
    timestamp uploaded_at
  }

  trip_state_log {
    uuid id PK
    uuid trip_id FK
    uuid changed_by FK
    varchar from_status
    varchar to_status
    timestamp changed_at
  }

  users ||--o{ trips : "owner_id"
  users ||--o{ trip_members : "user_id"
  trips ||--o{ trip_members : "trip_id"
  trips ||--o{ trip_days : "trip_id"
  trips ||--o{ user_trip_preferences : "trip_id"
  trips ||--o{ trip_state_log : "trip_id"
  trip_days ||--o{ activities : "trip_day_id"
  trip_days ||--o{ media_assets : "trip_day_id"
  trip_days ||--o{ day_budgets : "trip_day_id"
  user_trip_preferences ||--o{ day_budgets : "user_trip_pref_id"
  user_trip_preferences ||--o{ user_activity_states : "user_trip_pref_id"
  activities ||--o{ user_activity_states : "activity_id"
  activities ||--o{ media_assets : "activity_id"
  users ||--o{ media_assets : "uploaded_by"
  users ||--o{ trip_state_log : "changed_by"
```

### 7.2 Schema Design Decisions

**`trip_members`** — a join table instead of a simple `sharedWithUserIds` array. A relational database requires a separate table for many-to-many relationships. As a bonus, `joined_at` and `role` are available without an additional migration.

**`user_trip_preferences`** — an intermediate table between `users` and `trips`. Holds the total budget and default currency for a given user on a given trip. `day_budgets` and `user_activity_states` hang off it via `user_trip_pref_id` — this guarantees a single owner for that data and simplifies queries.

**`media_assets.activity_id`** — nullable. A photo can be assigned to a day in general (`trip_day_id`), to a specific activity (`activity_id`), or both.

**`trip_state_log`** — added from the start, since specification section 3.2 (Trip Lifecycle) explicitly suggests this mechanism in support of Tripper Wrapped. The cost of adding it now is zero; the cost of migrating it in v2.0 would be high.

**`available_currencies` on `trips`** — a `JSONB` column. Flexibility without a separate table, appropriate for a simple array of ISO 4217 strings.

### 7.3 Decisions Future-Proofing Later Versions

The following columns are added in the MVP deliberately, even though their business logic only lands in later phases. The cost of adding them now is zero — a column with a default value. The cost of skipping them is a migration on a table with production data.

**`trips.visibility` — defaults to `'private'` (preparation for v3.0)**

v3.0 requires distinguishing private and public trips (social feed, cloning). A column with a default of `'private'` means all existing trips are automatically private — correct behavior with no extra logic required.

```typescript
@Column({ default: 'private' })
visibility: 'private' | 'public';
```

**`deleted_at` on `users`, `trips`, `activities` — soft delete (preparation for v2.0/v3.0)**

Hard-deleting a record leads to orphaned data: media linked to a trip (v2.0), comments and likes from other users on a public trip (v3.0). Soft delete via a nullable `deleted_at` solves this — TypeORM handles it through `@DeleteDateColumn()`, which automatically filters `WHERE deleted_at IS NULL` on all queries. Physical data deletion (GDPR) is a separate, on-request operation, not the default behavior of `DELETE`.

```typescript
@DeleteDateColumn()
deletedAt: Date | null;
```

**`trip_members.role` — `'owner' | 'member'` (preparation for v1.x Phase 7)**

Without a `role` field, the only source of truth for trip ownership is the `owner_id` column on `trips`. That's sufficient for the MVP, but Phase 7 (collaboration) and a possible v3.0 (trip ownership transfer) require a more flexible permissions model. A `role` column on `trip_members`, with `'owner'` for the creator and `'member'` for participants, provides that flexibility without rewriting guard logic.

```typescript
@Column({ default: 'member' })
role: 'owner' | 'member';
```

**`activities.estimated_cost` + `estimated_cost_currency` (preparation for v1.x Phase 5)**

The `Activity` model in the specification doesn't include an estimated cost, but Phase 5 (budget) will need it to calculate the progress bar and compare planned vs. actual in the summary. Adding these two columns when the `activities` table is created in Phase 3 eliminates the need for a migration in Phase 5. Both are nullable — an activity with no cost specified is valid.

**`users.auth_providers` as `JSONB` instead of `varchar` (preparation for account linking)**

The current model assumes one account = one login method. Specification section 4.1.2 flags linking Google + email/password as an open topic. Changing the column from `varchar` to a `JSONB` array (`["local"]`, `["google"]`, `["local", "google"]`) is free now and avoids a destructive migration once account linking is implemented.

### 7.4 Indexes

Beyond primary and unique keys, the following indexes are required:

| Table | Column | Reason |
|---|---|---|
| `trips` | `owner_id` | Filtering a user's trips on the dashboard |
| `trips` | `status` | Filtering by tab (upcoming/active/archived) |
| `trips` | `invite_code` | Lookup by invite code |
| `trip_members` | `(trip_id, user_id)` | Composite PK — membership uniqueness |
| `trip_days` | `trip_id` | Fetching a trip's days |
| `activities` | `trip_day_id` | Fetching a day's activities |
| `user_activity_states` | `user_trip_pref_id` | Fetching states for a user |
| `media_assets` | `trip_day_id` | Fetching a day's media |

---

## 8. API Contract

### 8.1 General Conventions

- **Prefix:** all endpoints under `/api/v1/`
- **Error response format** — a uniform shape for all errors:

```json
{
  "statusCode": 404,
  "error": "Not Found",
  "message": "Trip not found"
}
```

- **Pagination** (for endpoints that may return long lists): query params `?page=1&limit=20`, response wraps the data as:

```json
{
  "data": [...],
  "meta": { "page": 1, "limit": 20, "total": 47 }
}
```

- **Dates** — always ISO 8601 (`2026-07-15T10:30:00Z`)
- **Currencies** — always ISO 4217 (`PLN`, `EUR`)
- **UUIDs** — used in every response instead of auto-increment IDs

### 8.2 Endpoints — Overview

#### Auth

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/v1/auth/register` | Register with email + password |
| `POST` | `/api/v1/auth/login` | Log in, returns access + refresh token |
| `POST` | `/api/v1/auth/refresh` | Refresh the access token |
| `POST` | `/api/v1/auth/logout` | Revoke the refresh token |
| `GET` | `/api/v1/auth/google` | Redirect to Google OAuth |
| `GET` | `/api/v1/auth/google/callback` | Google OAuth callback |
| `POST` | `/api/v1/auth/complete-profile` | Complete profile after Google OAuth (set username) |

#### Users

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/v1/users/me` | Logged-in user's profile |
| `PATCH` | `/api/v1/users/me` | Update profile (displayName, avatarUrl) |
| `GET` | `/api/v1/users/check-username/:username` | Check username availability (debounced) |

#### Trips

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/v1/trips` | List the user's trips (owned + as participant) |
| `POST` | `/api/v1/trips` | Create a new trip |
| `GET` | `/api/v1/trips/:id` | Trip details with days and activities |
| `PATCH` | `/api/v1/trips/:id` | Update trip metadata |
| `DELETE` | `/api/v1/trips/:id` | Delete a trip (owner only) |
| `PATCH` | `/api/v1/trips/:id/status` | Change state (draft→upcoming→active→archived) |
| `POST` | `/api/v1/trips/:id/join` | Join a trip via invite code |
| `DELETE` | `/api/v1/trips/:id/leave` | Leave a trip (participant) |

#### Activities

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/v1/trips/:id/days/:dayId/activities` | Add an activity |
| `PATCH` | `/api/v1/trips/:id/activities/:activityId` | Update an activity |
| `DELETE` | `/api/v1/trips/:id/activities/:activityId` | Delete an activity |
| `PATCH` | `/api/v1/trips/:id/activities/reorder` | Reorder (drag & drop) — batch `sort_order` update |

#### Budget (User Preferences)

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/v1/trips/:id/preferences` | Logged-in user's preferences and budget for the trip |
| `PUT` | `/api/v1/trips/:id/preferences` | Set/update the total budget and daily limits |
| `PATCH` | `/api/v1/trips/:id/activities/:activityId/state` | Update an activity's state (isVisited, actualCost, isAttending) |

#### Media

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/v1/trips/:id/days/:dayId/media` | Upload a photo/video (multipart) |
| `DELETE` | `/api/v1/trips/:id/media/:mediaId` | Delete media |

#### Maps (proxy)

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/v1/maps/autocomplete` | Place autocomplete (Google Places) |
| `GET` | `/api/v1/maps/directions` | Route between points (Google Directions) |

---

## 9. JWT Strategy

### 9.1 Two Tokens: Access + Refresh

| Token | Lifetime | Stored Where |
|---|---|---|
| Access token | 15 minutes | Application memory (not localStorage, not a cookie) |
| Refresh token | 7 days | `httpOnly` cookie (inaccessible to JS) |

**Rationale:**
- Access token in memory (not localStorage) — not vulnerable to XSS. The short lifetime limits the attack window to 15 minutes.
- Refresh token in an `httpOnly` cookie — JavaScript can't read it, so XSS can't steal it. Sent automatically by the browser with every request to `/api/v1/auth/refresh`.

### 9.2 Refresh Flow

```
1. Client sends a request with the access token
2. Backend returns 401 (token expired)
3. The client's auth interceptor catches the 401
4. Sends POST /auth/refresh (cookie sent automatically)
5. Backend verifies the refresh token, returns a new access token
6. The interceptor retries the original request with the new token
7. If the refresh also expired → logout, redirect to login
```

### 9.3 Refresh Token Storage in the Database

Refresh tokens are stored in a `refresh_tokens` table (separate, not included in the ERD above — auth-only):

```
refresh_tokens
  id          uuid PK
  user_id     uuid FK
  token_hash  varchar   ← hash of the token, never stored raw
  expires_at  timestamp
  created_at  timestamp
  revoked_at  timestamp (nullable)
```

On logout — `revoked_at` is set to `NOW()`. On refresh — the backend checks whether the token has been revoked or has expired.

### 9.4 Google OAuth — Flow with Profile Completion

```
1. GET /auth/google → redirect to Google
2. Google callback → /auth/google/callback
3. Backend checks whether the email already exists in users
   a. Exists → standard login (access + refresh token)
   b. Doesn't exist → temporary token (5 min) with Google data (email, name, avatar)
                     → client redirects to /auth/complete-profile
4. POST /auth/complete-profile with the temporary token + username
5. Backend creates the User record → returns access + refresh token
```

The temporary token from step 3b is single-use — once consumed in complete-profile, it's invalidated.

---

## 10. NestJS Code Conventions

### 10.1 Input Validation

Every DTO uses `class-validator` decorators. A global `ValidationPipe` is configured in `main.ts`:

```typescript
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,        // strips fields not present in the DTO
  forbidNonWhitelisted: true, // error when an unknown field is sent
  transform: true,        // automatic type transformation
}));
```

### 10.2 Error Format — Global Filter

```typescript
// common/filters/http-exception.filter.ts
@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const response = host.switchToHttp().getResponse();
    const status = exception.getStatus();
    response.status(status).json({
      statusCode: status,
      error: exception.name,
      message: exception.message,
    });
  }
}
```

### 10.3 Trip Invite Code

Format: 8 uppercase alphanumeric characters, e.g. `TRIP4X9Z`. Generated via `nanoid` when a trip is created. Uniqueness enforced via a `UNIQUE` constraint on the `invite_code` column. Doesn't expire — the owner can reset it (generates a new one) at any time.

### 10.4 Activity Reordering (Drag & Drop)

The `PATCH /activities/reorder` endpoint accepts an array of `{ id, sortOrder }` pairs and updates them in a single transaction:

```typescript
// Payload
{ "activities": [{ "id": "uuid1", "sortOrder": 0 }, { "id": "uuid2", "sortOrder": 1 }] }
```

The backend writes the entire array within a transaction — either all positions are updated, or none are.

---

## 11. Environment Variables (`.env.example`)

```env
# Server
PORT=3000
NODE_ENV=development

# Database
DATABASE_URL=postgresql://tripper:tripper@localhost:5432/tripper

# Redis
REDIS_URL=redis://localhost:6379

# JWT
JWT_ACCESS_SECRET=change-me-access
JWT_REFRESH_SECRET=change-me-refresh
JWT_ACCESS_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d

# Google OAuth
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_CALLBACK_URL=http://localhost:3000/api/v1/auth/google/callback

# Google Maps
GOOGLE_MAPS_API_KEY=

# S3 / Object Storage
S3_ENDPOINT=http://localhost:9000
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
S3_BUCKET=tripper-media
S3_REGION=auto

# Frontend URL (CORS)
FRONTEND_URL=http://localhost:4200
```

---

## 12. Local Development Environment

All services run via **Docker Compose** — zero manual installs beyond Node.js.

```yaml
# docker-compose.yml (skeleton)
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: tripper
      POSTGRES_USER: tripper
      POSTGRES_PASSWORD: tripper
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    ports:
      - "9000:9000"   # S3 API
      - "9001:9001"   # Web console
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
```

---

## 13. Repository Strategy

Frontend and backend live in **separate repositories** (`tripper-frontend`, `tripper-api`). Shared types are obtained by generating an OpenAPI client on the frontend side, not via a shared npm package.

The NestJS backend, decorated with `@ApiProperty()`, exposes `openapi.json` at `/api-json`. The frontend regenerates its API client from this contract whenever it changes. If the backend changes the contract and the frontend isn't regenerated, TypeScript catches the mismatch at compile time.

---

## 14. Backend Roadmap

Split into major versions, with v1.0 broken down into phases. Each phase/version ends with something genuinely usable — none requires finishing everything at once.

### v1.0 — Core (MVP)

| Phase | Scope |
|---|---|
| **Phase 0** | Foundation: backend skeleton (NestJS) + database + one test endpoint + connection from the frontend. Goal: get comfortable with request → response → database before adding domain complexity. |
| **Phase 1** | Auth: registration (email+password, Google OAuth), JWT (access + refresh), `AuthGuard`. Full `User` model per section 4.1. |
| **Phase 2** | Trip CRUD: `Trip` and `TripDay` entities. States limited for now to `draft`/`upcoming` (manual transitions). |
| **Phase 3** | Activities and timeline: `Activity` entity, add/edit/delete, reordering support (drag & drop endpoint), day selection. *The heart of the MVP — first point of real usability.* |
| **Phase 4** | Map and routes: Google Places API (autocomplete, locations), markers + route (Polyline) data. |

🏁 **End of v1.0:** planning + timeline + map, with a user account and cloud persistence from day one.

### v1.x — Feature Expansion

| Phase | Scope |
|---|---|
| **Phase 5** | Budget (solo): `UserTripPreferences`, `DayBudget`, estimated costs, budget usage progress bar. Multi-currency support introduced (section 4.3). |
| **Phase 6** | Live mode: manual `active` switch (or automatic per `autoStartEnd`), visited-places checklist, actual costs on the fly, free-form plan editing. |
| **Phase 7** | Group collaboration: access code, shared skeleton (`sharedWithUserIds`), Opt-Out mechanism (section 3), budget isolation between participants. |

### v2.0 — Memories and Archive

`archived` status, photo/video upload (`MediaAsset`), "Storyline" mode, summary statistics (financial/logistics charts, Tripper Wrapped, carbon footprint).

### v2.x — Album Generator

Export/formatting of a photobook from collected memories. Export format (PDF or other) to be decided closer to implementation.

### v3.0 — Social Layer *(distant future, direction without details)*

A separate product category — public, social, with different requirements (privacy, moderation, social graph) than the core app:

- Per-trip social feed
- Likes, comments, user profile
- Public/private trips (requires the `visibility: 'private' | 'public'` field on `Trip`)
- Cloning a public trip as inspiration for one's own plan
- **Age verification required before rollout** — see section 4.1.3

Deliberately left undefined at this stage — a topic to develop once the phase is actually approaching.

---

## 15. Open Questions / Deferred Topics

Collected in one place to avoid losing track of them:

- **Album generator** — generation model (client/CSS print vs. server) to be decided in v2.x.
- **Media upload limits** (size, count, formats) — to be decided when designing storage in v2.0.
- **Cloning another user's trip (v3.0)** — copy scope (structure only vs. photos/notes) undefined.
- **`authProvider` as a single value vs. a list** — currently one account = one login method; account linking has not yet been designed.
- **Currency exchange conversion** — deferred until there's a real need (section 4.3).
- **Trip state transition log/history** — useful for future Wrapped statistics, not yet formally modeled.

---

*This document is the single source of truth for the Tripper backend (`tripper-api`). The frontend (`tripper-frontend`) maintains its own architecture documentation covering UI screens, design system, and Angular-specific concerns.*

