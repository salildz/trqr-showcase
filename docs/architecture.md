# TRQR — Architecture Overview

## System Architecture

TRQR is a single-tenant SaaS product deployed as a set of Docker containers behind Cloudflare and Nginx.

```
Internet
  │
  ▼
Cloudflare (CDN, DDoS protection, Turnstile CAPTCHA)
  │
  ▼
Nginx (TLS termination, reverse proxy)
  │
  ├── /* ──────────────▶ React SPA (Vite, static files)
  │
  └── /api/* ──────────▶ Express API (Node.js 22)
                              │
                    ┌─────────┼──────────┐
                    ▼         ▼          ▼
               PostgreSQL   Redis     File Storage
               (primary DB) (cache /  (menu item
                            token     images)
                            blacklist)
```

---

## Frontend

The client is a **React 19 SPA** built with Vite. It communicates with the API exclusively over HTTPS using Axios with an interceptor layer for token refresh and error normalization.

**Key design decisions:**

- **MUI v7** provides the entire design system. Dashboard components (cards, tables, dialogs) are assembled from a small set of reusable primitives: `DashboardSurfaceCard`, `DashboardActionStack`, `DashboardStateCard`.
- **Plan enforcement is duplicated client-side** — not for security (server enforces), but for UX: plan-limit warnings and upgrade prompts are shown before a request is even made.
- **i18next** handles all string externalization. Turkish is the primary language; English is a full translation.
- **dnd-kit** powers the menu editor's drag-and-drop for category and item reordering.
- **jsPDF + html2canvas** power the print export feature — the selected QR tabletop template is rendered to canvas and exported as a PDF.

---

## Backend

The API is an **Express 5** application running on Node.js 22. All business logic is organized into controller files grouped by domain.

### Middleware Stack (in execution order)

```
Request
  │
  ├─ extractRealIp          – Extract actual client IP from Cloudflare CF-Connecting-IP header
  ├─ requestIdMiddleware     – Assign UUID to req.requestId; set X-Request-ID response header
  ├─ helmet                  – HTTP security headers (CSP, HSTS, X-Frame-Options, etc.)
  ├─ express.json(16kb)      – Parse JSON body; tight global limit
  ├─ cookieParser            – Parse HTTP-only cookies (refresh token)
  ├─ hpp                     – HTTP Parameter Pollution protection
  ├─ xssSanitize             – Recursive XSS sanitization on body / query / params
  ├─ cors                    – Whitelist-based origin validation
  ├─ globalApiLimiter        – 600 req / 15 min rate limit across all /api routes
  │
  └─ Route-specific middleware
       ├─ authenticateToken          – Verify JWT; check Redis token blacklist
       ├─ loadPlan                   – Attach restaurant plan config to req
       ├─ requireFeature             – Gate access by plan feature flag
       ├─ requireMenuLimits          – Validate menu mutations against plan limits
       ├─ requireAdmin               – Role check (user.role === 'admin')
       ├─ verificationResendLimiter  – 5 resend requests / hour / IP
       └─ validate(schema)           – Zod schema validation
```

### Domain Controllers

| Controller | Responsibility |
|------------|---------------|
| `userController` | Register, login, token refresh, logout, password reset, email verification (verify / resend / change pre-verification) |
| `menuCrudController` | Menu read/write with before/after diff audit logging |
| `menuQrController` | QR code generation, template management, print export |
| `menuMetaController` | Restaurant name, language, established year updates |
| `menuImageController` | Menu item image upload and removal |
| `statsController` | Analytics aggregations (views, revenue, top items, hourly) |
| `tableController` | Table CRUD, order accumulation, table merge, bill close |
| `restaurantController` | Restaurant settings, feature flags |
| `adminController` | Platform admin: user/restaurant management, audit log queries |

---

## Database

**PostgreSQL 16** via Sequelize 6 ORM.

### Core Models

```
User
  ├── id (UUID)
  ├── username, email, passwordHash
  ├── role (user | admin)
  ├── onboardingState, lastLoginAt
  ├── emailVerified (BOOLEAN, default false)
  ├── emailVerifiedAt (TIMESTAMP, nullable)
  ├── emailVerificationTokenHash (VARCHAR 64, nullable)  ← SHA-256 of raw token
  ├── emailVerificationTokenExpiry (TIMESTAMP, nullable) ← 24h TTL
  └── verificationEmailSentAt (TIMESTAMP, nullable)      ← 60s resend cooldown

Menu
  ├── id (UUID)
  ├── restaurantPublicId  ──────────────── index
  ├── restaurantName
  ├── menuData (JSONB)           ← categories → items tree
  ├── menuTemplate, menuLanguage
  └── establishedYear

Restaurant
  ├── id (UUID)
  ├── menuId (FK → Menu)
  ├── tableCount, plan
  └── featureFlags (JSONB)       ← per-restaurant plan overrides

Table
  ├── id (UUID)
  ├── restaurantId (FK → Restaurant)
  ├── tableNumber, isOccupied
  ├── orders (JSONB)             ← accumulated items before bill close
  └── mergedIntoTableId

SalesRecord
  ├── restaurantPublicId         ← denormalized for retention independence
  ├── tableNumber, items (JSONB)
  ├── totalAmount, paymentMethod
  └── closedAt

MenuAccessLog
  ├── restaurantPublicId
  ├── ipAddress, userAgent, referrer
  └── accessedAt

AuditLog (immutable — no updates ever)
  ├── actorId, actorUsername, actorRole
  ├── requestId (UUID, indexed)
  ├── httpMethod, httpPath
  ├── ipAddress, userAgent
  ├── action, resourceType, resourceId
  ├── status (success | failure)
  └── metadata (JSONB)          ← before/after diffs, error context
```

### Data Lifecycle

- **MenuAccessLog** — purged after `LOG_RETENTION_DAYS` (default 90 days) via nightly cron
- **SalesRecord** — purged after `SALES_RETENTION_DAYS` (default 365 days) via nightly cron
- **AuditLog** — never purged; append-only

---

## Authentication Flow

```
Client                          Server
  │                               │
  ├─ POST /api/users/register ──▶ │
  │  { username, email, password }│  hash password (bcrypt)
  │                               │  generate 256-bit token → SHA-256 hash stored
  │                               │  send verification email (non-blocking)
  │◀── 201 { email } ────────────│
  │                               │
  │  [user clicks link in email]  │
  │                               │
  ├─ GET /api/users/verify-email ▶│
  │  ?token=<raw-hex>             │  hash token → compare with stored hash
  │                               │  set emailVerified=true, clear token
  │◀── 200 OK ───────────────────│
  │                               │
  ├─ POST /api/users/login ──────▶│
  │  { email, password }          │
  │  [+ Turnstile token if        │  verify password (bcrypt)
  │    brute-force detected]      │  check emailVerified → 403 if false
  │                               │  check progressive auth limit
  │                               │
  │◀── 200 { accessToken } ───────│
  │    Set-Cookie: refreshToken   │  (HTTP-only, Secure, SameSite=Strict)
  │                               │
  │  ... subsequent requests ...  │
  │                               │
  ├─ GET /api/menu ─────────────▶ │
  │  Authorization: Bearer <jwt>  │  verify JWT signature + expiry
  │                               │  check Redis blacklist
  │◀── 200 { menu } ─────────────│
  │                               │
  │  ... token expires ...        │
  │                               │
  ├─ POST /api/users/refresh ───▶ │
  │  Cookie: refreshToken         │  validate refresh token
  │                               │  issue new access token
  │◀── 200 { accessToken } ───────│
  │                               │
  ├─ POST /api/users/logout ────▶ │
  │  Authorization: Bearer <jwt>  │  add JWT to Redis blacklist
  │                               │  clear refresh cookie
  │◀── 200 OK ───────────────────│
```

---

## Plan Enforcement

Plan limits are enforced **exclusively server-side** via middleware. The client replicates limits only for UX purposes (showing upgrade prompts before a request fails).

On every menu mutation:
1. `loadPlan` middleware fetches the restaurant's current plan config
2. `requireMenuLimits` validates the incoming payload against plan limits
3. If a limit is exceeded, the API returns `403 PlanLimitExceeded` with the specific feature, limit, and current count
4. The client shows a specific, actionable error message with a revert option

Feature gates (payment tracking, data export, analytics tier) use the same `requireFeature(flagName)` middleware.

---

## Audit Logging

Every meaningful server action emits an audit log entry with:

- **Who**: `actorId`, `actorUsername`, `actorRole` (denormalized at write time — survives user deletion)
- **Where**: `requestId` (UUID propagated from ingress to audit record), `httpMethod`, `httpPath`
- **From**: `ipAddress` (real IP after Cloudflare extraction), `userAgent`
- **What**: `action` (dot-notation, e.g. `menu.updated`), `resourceType`, `resourceId`
- **Result**: `status` (success / failure)
- **Context**: `metadata` (JSONB) — before/after diffs for mutations, error codes for failures

The `AuditLog` model is immutable: no `UPDATE` or `DELETE` operations are performed on it anywhere in the codebase.
