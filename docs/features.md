# TRQR — Feature Reference

## Menu Management

The menu builder is the core of the dashboard experience.

**Categories**
- Create, rename, reorder (drag-and-drop), and delete categories
- Live validation against plan limits (max categories, max items per category)
- Category order persisted immediately on drag

**Menu Items**
- Add items with: name, price, description (optional), photo (optional)
- Toggle item availability on/off without deleting
- Drag to reorder within a category
- Photo upload with plan-tier size limits (512 KB on Free, up to 2 MB on Pro+)

**Save Behavior**
- Single "Save" action for the full menu
- If a plan limit is exceeded after a plan downgrade, the save is blocked with a specific error message and a one-click revert option

---

## QR Code & Templates

**QR Code**
- Unique QR code generated per restaurant
- Encodes a public URL pointing to the restaurant's live menu — uses the custom slug URL if one is set, otherwise falls back to the UUID-based public URL
- Downloadable as PNG from the dashboard
- The "Go to Menu" link in the dashboard also reflects the custom slug when active
- **Copyable public menu link** — the QR page shows the public menu URL up front in a read-only field with a one-tap copy button ("Copied" confirmation), plus copy/open shortcuts, so the owner can paste it straight into a social bio, WhatsApp, or Google Business without digging through the design panel

**Menu Templates (6)**
- **Classic** — Warm serif design, ideal for traditional restaurants
- **Minimal** — Clean monochromatic look for modern cafés
- **Modern** — Bold gradient with vibrant accents
- **Rustic** — Earthy, textured feel for farm-to-table or casual dining
- **Elegant** — Dark luxury with gold typography, ideal for fine dining
- **Neon** — Cyberpunk-inspired dark theme with glowing accents

Template access is gated by plan tier. Upgrading immediately unlocks the template for use.
Template selection is configured on a dedicated Menu Designer page, separate from Restaurant Settings.

**Live demo**
- A sample menu is always available at `/menu/sample` — no login or restaurant account required
- Uses the Rustic template with a complete Turkish restaurant dataset including item photos
- A floating template-switcher bar lets visitors preview all 6 templates on the same sample data

**Custom Menu URL** *(Pro and Enterprise)*
- Set a short, memorable slug for the public menu (e.g., `trqr.net/menu/yildiz-restoran`)
- Real-time availability check with 600 ms debounce — instant feedback in the input field
- Slug validation: lowercase letters, digits, and hyphens; 3–50 characters; reserved word list enforced
- Both the UUID-based URL and the custom slug resolve to the same menu (custom slug takes priority)
- Slug is cleared automatically if the restaurant's plan drops below Pro after the grace period expires
- Locked with an upgrade prompt for Free and Starter plans

**QR Tabletop Card Designer**
- Each menu template has a matching QR tabletop card design
- Preview updates live as template is selected
- Export to PDF in three sizes: A5, A6, Square

---

## Table Management

- Create tables numbered sequentially
- **Custom table names** — each table can be given a custom display name (e.g., "Garden", "VIP 1", "Terrace") that appears everywhere in the dashboard: cards, order dialogs, merge/transfer pickers, and snackbar notifications. Defaults to "Table N" when no custom name is set.
  - Stored as `customName` on the `Tables` DB row; nullable (null = use default label)
  - Rename dialog accessible via the pencil icon on every table card; supports clearing the custom name to revert to the default
  - Name changes update local state optimistically and are confirmed via `PUT /api/tables/:tableId/name`
- When table count is reduced, tables are deleted from the highest-numbered end, preserving custom names for all remaining tables
- Mark tables as occupied / free
- Track per-table order accumulation
- Merge a secondary table into a primary (combined bill view)
- Close bill with payment method: Full payment / Equal split / Per-item

---

## Staff Accounts & Roles

Restaurant-scoped sub-accounts for waiters, kitchen, cashiers, and managers — completely separate from the billing owner account.

**Identity model**
- New `StaffUser` table, isolated from the `User` (owner) table; composite `(restaurantId, username)` unique
- 6-digit numeric PIN (bcrypt-hashed) for fast in-restaurant login — mobile devices auto-open the numeric keypad; optional full password for managers
- Distinct JWT type (`type: 'staff'`) and a separate refresh cookie (`staffRefreshToken`) so an owner and a staff member can stay signed in on the same physical device
- Per-device sessions with a "Devices" panel: the owner can revoke an individual device with one tap (added to Redis blacklist)
- Lockout after 5 failed PINs for 15 minutes; per-IP rate limit on login

**Table assignment mode** *(per-restaurant setting)*
- `free` (default) — every waiter sees every table, picks up whichever is free. Matches the most common small-restaurant workflow with zero setup.
- `assigned` — each table is assigned to one or more specific waiters; a waiter only sees their own tables. Managers and the owner still see everything. Server-side filter (not client-only), so SSE events also respect the assignment.
- Switching modes is a single toggle in the Staff tab; in `assigned` mode a multi-select table picker appears next to each waiter row.
- Long-press a table card in the waiter app to temporarily transfer it to a colleague (`tables.transfer` permission).

**Role + permission system**
- Four built-in roles: `manager`, `waiter`, `kitchen`, `cashier`
- Permission flags include `tables.takeOrder`, `tables.cancelItem`, `tables.merge`, `tables.transfer`, `payments.collect`, `payments.applyDiscount`, `kitchen.view`, `kitchen.markReady`, `kitchen.reprintTicket`, `staff.manage`, `reports.viewDay`
- Owner-side override panel (hidden behind an "Advanced" accordion so the default flow stays a single role-pick): every role has defaults, but a single user's flags can be fine-tuned (e.g., "waiter Mehmet can apply discounts, others can't", or "this waiter can take orders but not collect payments" for restaurants with a dedicated cashier)
- Contextual constraints (e.g., "waiters can only cancel items on tables they opened, within X minutes") enforced in the controller — not in the JSON, where they'd be brittle

**Approval settings (owner-only, Staff tab)**
- **Manager override PIN** — a restaurant-level 6-digit PIN that authorises over-threshold discounts/comps/voids from any staff terminal, so a restaurant with no manager-role staff still has a working approver (stored bcrypt-hashed; only "set / not set" is ever exposed)
- **Approval threshold (%)** — how much of a bill a non-manager may discount/comp/void on their own before a manager PIN is required; owner-tunable 0–100 (default 5, `0` = always require approval). Governs both the comp/void and the payment-discount gates

**Staff management UI (owner-only)**
- New "Staff" tab in the dashboard
- Create, edit, deactivate users; system-generated initial PIN displayed once with copy
- "Reset PIN" forces a change on next login (`mustChangePinAt`)
- Plan-tier seat limits enforced server-side: Starter 3, Pro 10, Enterprise 50

---

## Waiter App

A mobile-first, permission-gated slice of the dashboard purpose-built for phone and tablet use during service.

- Available on **Starter and above**; route lives under `/staff/waiter` after PIN login
- Bottom-nav layout (Tables · Open Orders · Profile)
- Take orders directly from the menu, add per-item notes ("no onion", "extra spicy"), use a quantity stepper, then **Send to Kitchen** in one batch
- Collect payments (full / equal split / per-item) when `payments.collect` is granted; discounts gated separately by `payments.applyDiscount`
- "Ready" banner with vibration when the kitchen marks one of the waiter's orders ready (via the SSE realtime channel)
- Device-pinning so the waiter doesn't need to re-enter the PIN every shift

---

## Kitchen Display System (KDS)

Real-time order feed targeted at a tablet or in-store screen in the kitchen.

- Available on **Pro and above**; route `/staff/kitchen`, optimised for landscape full-screen
- Cards appear automatically when a waiter sends an order (SSE-driven, no polling)
- Each card shows table number + custom name, waiter, line items with notes, and a live "preparing" timer
- **"Ready" button** marks the whole order ready, which clears the card and pushes a banner to the waiter's phone
- **Thermal receipt printing**:
  - Server renders the ticket as ESC/POS bytes (`esc-pos-encoder`)
  - **Pro**: WebUSB / WebSerial — the KDS browser sends the bytes directly to a USB or serial thermal printer (58 mm / 80 mm)
  - **Enterprise** *(planned)*: optional "TRQR Print Agent" — a small Node binary running on a Raspberry Pi or mini-PC in the restaurant LAN, queues jobs and handles network/USB printing offline-safely
  - Reprint button (gated by `kitchen.reprintTicket`) re-emits the same content with a `[REPRINT]` stamp
  - Offline queue: pending tickets persisted in localStorage and flushed when the printer reconnects
- **Multiple kitchen stations** *(Enterprise, planned)*: menu items will be taggable with a station (`grill`, `cold`, `bar`, `dessert`) and routed to separate printer queues; the per-item station column is already in place server-side

---

## Realtime & Order State Machine

The plumbing that makes the waiter app and KDS feel live.

**Order item state machine**
- Each line item carries a stable UUID (replaces today's array-index identity, which races under concurrent edits)
- States: `pending` → `preparing` → `ready` → `served`; `cancelled` is terminal from any state
- `pending` items are visible to the waiter but not yet to the kitchen — they batch on "Send to Kitchen"
- Existing open tables backfilled to `served` during migration so nothing dumps onto the kitchen screen on day one

**Realtime channel — Server-Sent Events**
- `GET /api/realtime/stream?channel=kitchen|tables` — auth-gated, restaurant-scoped, heartbeat every 25 s
- Event types: `order.created`, `order.itemAdded`, `order.itemCancelled`, `order.statusChanged`, `order.sentToKitchen`, `table.merged`, `table.closed`, `kitchen.ticketReprinted`
- Single Node `EventEmitter` per process now; planned move to Redis Pub/Sub when we add a second backend instance
- EventSource on the client with a 3 s polling fallback for transient network blips

**Why SSE, not WebSockets**
- Traffic is one-way (server → client); waiter / KDS actions are plain REST mutations
- Survives Cloudflare / Nginx by default with the right `Cache-Control` and `X-Accel-Buffering: no` headers
- Built-in reconnect; no custom heartbeat / protocol layer needed

---

## Payment Tracking

Available on Starter and above.

- Record sales when a table's bill is closed
- Payment methods: full, split, per-item
- **Percentage discount** — apply a 0–100% discount at the payment confirmation step; raw subtotal, discount line, and discounted total shown in the breakdown
- Items and totals stored per transaction in `SalesRecord`, including `discountPercent`, `discountAmount`, and `splitType`
- Used as input for the Analytics revenue charts

---

## Comp & Void (İkram / İptal-İade)

Available on Starter and above (reuses the `payments.applyDiscount` permission).

Item-level bill adjustments for the real-world "there's a hair in this dish" / wrong-item / customer-return cases — works even on an **already-served** line, which the plain pre-kitchen cancel cannot touch.

- **Two types** — *comp* (İkram: complimentary, food was made/served but not charged) and *void* (İptal-İade: removed as an error/return), tracked separately for reporting
- **Quantity-aware** with a **mandatory reason**; available both on the owner dashboard (table cart) and the waiter table-detail screen
- Removed lines stay **visible on the bill** (struck through, labelled İkram/İade) for guest + staff transparency, but are excluded from every total, payment picker and "amount owed"
- **Manager-approval gate** — gated by `payments.applyDiscount`; a non-manager may comp/void up to a **configurable threshold** of the bill on their own (default 5%, owner-tunable 0–100% from Staff settings; `0` = every comp/void needs approval), beyond which a **manager PIN** is required (verified server-side — the POS "swipe a manager" pattern). With a valid PIN there's no upper ceiling; owners and managers act directly. The same configurable threshold + manager-PIN gate also governs payment-step **discounts**. Two valid approvers: any active manager's login PIN, **or** an owner-set restaurant **override PIN** (Staff settings) so a restaurant with no manager-role staff still has a working approver.
- **Brute-force-hardened PIN** — the manager-approval PIN has its own per-`(restaurant, staff)` lockout (5 wrong PINs → 15-minute cool-off), refused *before* the bcrypt check and degrading open if Redis is down. Failed and locked-out attempts are audit-logged (`manager.approval.failed` / `.lockedOut`); a `429` carries a `Retry-After`. The UI then guides the waiter to have a manager sign in with their own account.
- **Idempotent** — each adjustment request carries an idempotency key, so a double-tap or network retry replays the original result instead of comping the line twice.
- Every adjustment is written to an **append-only `OrderAdjustment` ledger** (item, amount, type, `fromStatus`, reason, staff, **who approved it** — manager id or "override" — and timestamp) — the foundation for comp/void/waste reporting and a future Reports page. The existing pre-kitchen cancel feeds the same ledger so food-waste reporting is complete.
- Backed by a generalised order-item state machine: terminal `comped` / `voided` statuses alongside `cancelled`, all part of the shared `NON_BILLABLE` set

---

## Receipt History

Available on Starter and above.

Each completed payment generates a uniquely numbered receipt (`YYYYMMDD-NNNN`, sequential per restaurant per day). The Receipts dashboard page provides:

**Filters**
- Date range (from / to)
- Payment method (cash / card / all)

**Receipt list**
- Paginated table (20 per page) with receipt number, table, items, total, payment method, date
- Click any row to open an itemised detail dialog
- Per-receipt PDF export (80mm thermal receipt format)

**Plan enforcement**
- Receipts and Day Summary pages are visible to all plans, including the free tier
- Users without a qualifying plan see a blur-lock overlay with a clear upgrade prompt and a one-click path to the Subscription page
- The API returns 403 if the plan does not include `paymentTracking`; the UI handles this gracefully without an error page

---

## Day Summary Report

Available on Starter and above.

Select any calendar date and load an aggregate breakdown:

| Metric | Description |
|--------|-------------|
| Order Count | Number of closed bills for the day |
| Total Revenue | Sum of all `totalAmount` values |
| Cash Revenue | Revenue from cash payments |
| Card Revenue | Revenue from card payments |
| Total Discount | Sum of all `discountAmount` values |
| Comps (İkram) | Value comped off bills, from the `OrderAdjustment` ledger |
| Voids (İptal-İade) | Value voided/returned off bills |
| Waste (Fire) | Value of items that had reached the kitchen (`fromStatus` ≠ pending) and were never sold |
| Avg Order Value | Total revenue ÷ order count |

**Top Items table** — items ranked by quantity sold for the day.

**Top Adjustments table** — comped / voided / cancelled items ranked by value, with their type.

**PDF export** — generates an A4-format summary report via jsPDF + html2canvas.

---

## Analytics

Date range selector: Today / This Week / This Month / Custom (max 365 days).

**Summary Cards**
- Total menu views in range
- Unique visitors (distinct IP count)
- Total orders (closed bills)
- Total revenue

**Charts**
- Daily Views — bar chart by day
- Daily Revenue — area line chart by day
- Top Selling Items — horizontal bar chart ranked by quantity
- Hourly Distribution — access count by hour of day

Analytics tier (Basic / Standard / Advanced / Premium) gates the level of detail available per plan.

---

## Public Menu — Branding

Every public menu includes a "Powered by TRQR" watermark footer by default. The watermark:

- Is styled to match each template's color palette (dark on light themes, light/accent on dark themes)
- Is automatically hidden for Pro and Enterprise restaurants (`removeBranding` feature flag)
- Is never shown in the Dashboard print preview or Menu Designer preview mode
- Is rendered server-side per request — no client-side override is possible; the `removeBranding` flag is resolved from the restaurant's active plan on the backend

---

## Print Export

- Select template, paper format (A5 / A6 / Square)
- Preview renders live in browser
- Export to PDF via jsPDF + html2canvas pipeline
- Menu items with images show photo thumbnails in all templates

---

## Admin Panel

Platform administrators have access to a separate panel at `/admin`.

**Overview**
- Platform totals: restaurants, users, menu views, revenue
- Plan distribution breakdown

**Restaurant Management**
- List all restaurants with search (name, email, public ID)
- Change plan tier per restaurant
- Override feature flags independently of plan
- Edit restaurant name
- Delete restaurant (transactional cleanup of all associated data)

**User Management**
- List all users with search
- Change user role
- Reset password
- Delete user

**Audit Logs**
- Filter by: action, actor (partial match), IP address, resource type, resource ID, status, date range
- Paginated results
- Detail view per log entry: full HTTP context, before/after diff in metadata, actor info

---

## Onboarding

New restaurants are guided through a dismissible checklist:
1. Set restaurant name
2. Add your first menu item
3. Preview your public menu
4. Download your QR code

Each step tracks completion state independently. Checklist can be dismissed once all steps are complete.

---

## Security

### Email Verification

All accounts must verify their email address before they can log in.

**Verification flow**
1. User registers → verification email sent immediately
2. Account is hard-blocked from login until the email link is clicked
3. Stale refresh token cookies for unverified accounts are also rejected — no session-based bypass
4. Link contains a 256-bit random token (raw hex in email, SHA-256 hash stored in DB); expires in 24 hours

**Resend & email change before verification**
- Resend button with 60-second per-user cooldown (DB-tracked) and 5/hour IP rate limit
- Email address can be changed before verification — requires password confirmation; issues a new token

**Email delivery**
- Provider-agnostic service layer (`smtp` or `mailjet`)
- Default: Google Workspace SMTP via Nodemailer
- Bilingual HTML template — Turkish / English / bilingual mode based on user's browser language

### Email Change (Authenticated)

Verified users can change their registered email address from the Restaurant Management page.

**Flow**
1. User enters new email + current password in the "E-posta Değiştir" form
2. Password is verified server-side; new address is checked for uniqueness
3. A confirmation email (bilingual HTML with logo) is sent to the **new** address with a signed token (SHA-256 hash stored, 24 h TTL)
4. Until confirmed, a dismissible "pending change" banner is shown in the dashboard with a cancel option
5. On confirmation, `user.email` is atomically updated; a final uniqueness check guards against race conditions (another user registering the same address between request and confirm)
6. If the token has expired or the address was taken, the confirmation page shows a clear error with a path to request a new link

**Security details**
- 60 s per-user resend cooldown (DB-tracked `emailChangeSentAt`)
- Hashed token storage — raw token never persisted
- Clearing all pending-change fields on cancel or successful confirm
- Edge case: if the address is claimed between request and confirmation, the pending change is cleared and `EmailNowTaken` is returned

---

### Authentication

- **JWT Access Tokens** — 15-minute expiry, verified on every protected request
- **HTTP-only Refresh Cookies** — 7-day rotating token; `Secure`, `SameSite=Strict`
- **Redis Token Blacklist** — Revoked tokens blacklisted by JTI; checked on every authenticated request
- **Progressive CAPTCHA** — Cloudflare Turnstile triggered after repeated failed login attempts
- **bcrypt Password Hashing** — Passwords hashed before storage; never stored in plaintext
- **Rate Limiting** — Per-IP progressive limiter on auth endpoints; global 600 req/15 min across all API routes
- **Forgot Password** — Secure token-based reset flow:
  - 256-bit random token; SHA-256 hash stored in DB — raw token never persisted
  - 1-hour TTL; one-time use (cleared immediately on consumption)
  - Per-account 60-second resend cooldown; per-IP 5/hour rate limit
  - All active sessions (refresh tokens) invalidated on successful reset
  - Bilingual TR/EN reset email via SMTP
  - No user-enumeration — endpoint always responds with 200 regardless of whether the email exists

---

## Trial & Plan Lifecycle

Every new account starts on the **Starter plan** with a **5-day free trial**. No credit card is required.

**Trial period**
- New registrations are placed on the Starter plan with `trialEndsAt = registration time + 5 days`
- Full Starter feature access during the trial (15 tables, 4 templates, payment tracking, etc.)
- A countdown banner is shown in the dashboard during the trial
- Existing users registered within 5 days before the feature launched are automatically backfilled into the trial via a one-time migration

**Trial expiry → Grace period**
- When the trial ends (detected lazily on next `GET /api/restaurant` call, or by a daily cron job at 03:15 UTC), the plan is set to `free` and a **3-day grace period** begins
- A prominent warning banner appears on every dashboard tab during the grace period, showing the enforcement date and an "Upgrade Plan" shortcut
- No data is modified during the grace period — the user retains all their data

**Grace period feature access**
- During the grace period, all feature checks (`requireFeature`, `requireAnalyticsTier`) are bypassed — the restaurant retains full operational access to paid features (payment tracking, analytics tiers, etc.) until enforcement day
- Rationale: data cleanup has not happened yet, so blocking operations like payment recording during this window would be harmful
- The `loadPlan` middleware sets `req.isInGracePeriod = true` when `gracePeriodEndsAt` is in the future; each middleware guard checks this flag before enforcing plan limits

**Grace period enforcement**
- After the 3-day grace period (again, lazily on next request or via cron), free-plan limits are applied to the restaurant's data:
  - **Tables** — `tableCount` clamped to 5; excess non-merged tables deleted from the database
  - **Categories** — sorted by display order; first 5 kept, the rest deleted
  - **Items per category** — sorted by display order; first 15 per category kept
  - **Item images** — counted globally across all categories; `imageUrl` removed from items beyond the 12-image limit (images stripped from the highest-order items first)
  - **Menu template** — if the active template (e.g. Modern, Rustic, Elegant, Neon) is not available on the free plan, it is reset to `classic`
  - **Custom menu slug** — if the restaurant had a custom URL slug set (Pro feature), it is cleared; the menu reverts to the UUID-based public URL

---

## SEO & Marketing

- **Dynamic SEO Landing Pages**: Multiple dedicated landing pages (QR Menu, Digital Menu, Contactless Menu) optimized for search engines in both Turkish and English.
- **Service & Product Pages**: Specialized pages detailing pricing, system features, and example templates.
- **Dynamic Meta Tags**: Full Open Graph and Twitter Card support on all marketing pages via React Helmet Async.
- **Lead Generation**: Contact pages with fully integrated contact forms for enterprise inquiries.

---

## Blog System

A lightweight, fully integrated blog system for content marketing and user education.

- **Blog Index**: Paginated list of articles with featured images, excerpts, and reading times.
- **Article Pages**: Full Markdown rendering for rich blog posts.
- **Bilingual Content**: Separate articles for Turkish (`/blog`) and English (`/en/blog`) audiences.
- **SEO Optimized**: Each post includes custom meta tags, canonical URLs, and structured data ready for search engines.

---

## Restaurant Settings

Configurable per restaurant from the Restaurant Management page:

| Setting | Description |
|---------|-------------|
| **Restaurant Name** | Display name shown on the public menu header |
| **Table Count** | Number of active tables; clamped to plan limit |
| **Established Year** | Optional founding year shown on supported menu templates |
| **Currency** | Display currency for all prices (see below) |
| **Custom Menu URL** | Short slug for the public menu (Pro and Enterprise) |
| **Email** | Change the registered email address with confirmation flow |
| **Password** | Change account password |

---

## Multi-Currency Support

Restaurants can configure the currency used throughout the platform — in the dashboard, on the live public menu, on receipts, in analytics totals, and in print exports.

**Supported currencies**

| Code | Symbol | Name |
|------|--------|------|
| TRY | ₺ | Turkish Lira (default) |
| USD | $ | US Dollar |
| EUR | € | Euro |
| GBP | £ | British Pound |

**Where the currency symbol appears:**
- All price columns in Table Management (order accumulation and bill view)
- Payment steps — totals, subtotals, discount breakdown
- Receipt PDF exports
- Day Summary Report — revenue KPIs and top-item revenue
- Analytics — revenue cards and charts
- Print Menu export — item prices in all 6 templates
- Public guest menu — item prices across all 6 templates

**Implementation:**
- Currency stored as a `VARCHAR(8)` column (`currency`) on the `Menu` model; default `TRY`
- Updated via `PUT /api/menu/currency` (authenticated, Zod-validated)
- Frontend: `CurrencyContext` (React context + `useCurrency()` hook) provides `currencyCode` and `currencySymbol` to all dashboard and payment components; initialised from the menu fetch on Dashboard mount
- Public menu: `MenuCurrencyContext` (inline React context within `Menu.tsx`) resolved from the `getMenuByPublicId` response

---

## Internationalization

Every user-facing string in the dashboard, public menu, and error messages is localized in:
- **Turkish** (primary language)
- **English**

Language is set per restaurant and can be changed at any time from the dashboard settings. The language selection also affects the public guest menu.
