# TRQR — Feature Reference

## Menu Management

The menu builder is the core of the dashboard experience.

**Categories**
- Create, rename, reorder (drag-and-drop), and delete categories
- Live validation against plan limits (max categories, max items per category)
- Category order persisted immediately on drag
- **Mark an entire category as out of stock ("86")** in one toggle — every item under it shows as unavailable on the public menu and is blocked from new orders, without editing each item

**Menu Items**
- Add items with: name, price, description (optional), photo (optional)
- **Dietary tags** — a closed set of badges (vegan, vegetarian, gluten-free, lactose-free, contains-nuts, and one of three spice levels) picked as chips in the item dialog. Spice levels are mutually exclusive, and the server normalises the list on every write (unknown values dropped, duplicates collapsed, the hottest level kept) so a hand-edited menu can never render two chilli badges
- **English translation** — optional English name and description per item, in a collapsible section. Fallback is per field, not per item: a translated name with an untranslated description reads as English name + Turkish description, so a half-translated menu has no gaps
- **Mark a single item out of stock ("86")** without deleting it — unavailable items are greyed out on the public menu and blocked from new orders; a stale client that still tries to order one is rejected server-side
- Drag to reorder within a category
- Photo upload with plan-tier size limits (512 KB on Free, up to 2 MB on Pro+)

**Save Behavior**
- Single "Save" action for the full menu
- If a plan limit is exceeded after a plan downgrade, the save is blocked with a specific error message and a one-click revert option

---

## Guest Menu Experience

What a customer gets after scanning the QR. Everything here is available on **every plan, Free included**, and works identically across all six templates — the behaviour lives in one shared layer rather than being reimplemented per design.

**Search**
- Appears automatically once a menu passes 12 items; below that the whole menu is a scroll away and a search field would just be chrome
- **Turkish-aware matching**: the query and the menu text are lowercased with the Turkish locale *and* folded onto ASCII (ı/ş/ğ/ü/ö/ç), which fixes both directions of the classic trap — "cig kofte" finds "Çiğ Köfte" from a keyboard with no Turkish letters, and "ispanak" matches an item written "ISPANAK" (whose Turkish lowercase is the dotless "ıspanak")
- Every whitespace-separated token must match, so word order is irrelevant; an item matches on its name, description, English translation or its category name, and a category-name match surfaces that whole section
- Results render as one flat list with category headings — which is what makes search meaningful on the tab-based templates, whose normal layout only ever shows a single category
- Out-of-stock items stay in the results, greyed as everywhere else, rather than quietly disappearing

**Dietary badges**
- Vegan, vegetarian, gluten-free, lactose-free, contains-nuts, and a spice level (mild / spicy / very spicy)
- Drawn from `currentColor` so each template picks them up in its own palette; contains-nuts uses a caution mark, since it is the only allergen *warning* in the set
- Normalised again at render time, so legacy or hand-edited data can never show two spice levels or an unknown tag

**Item detail sheet**
- Tapping a row opens a bottom sheet with the large photo, full description, badges, price and the out-of-stock state
- Only rows that have a photo or a description are tappable — a bare name-and-price row doesn't open a sheet that repeats itself
- Routeless by design (client-side state), so the SEO prerender manifest and sitemaps are untouched. Tappable rows are proper buttons: Enter/Space open them and they take a focus ring

**TR/EN switch**
- Appears in the menu header only when at least one item actually carries an English translation — otherwise "English" would be Turkish content under English chrome, which reads as broken rather than bilingual
- The restaurant's own menu language is the default; a guest's choice is remembered **per restaurant** in local storage, so reading one menu in English doesn't switch every other menu that phone opens
- Page metadata and structured data keep following the restaurant's published language, so the JSON-LD can't end up claiming English over Turkish item names

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

## AI Menu Import *(Starter and above)*

Typing a full menu in by hand is the single biggest piece of friction at signup. This turns a photo of a printed menu into an editable draft.

- **Upload** up to 10 photos (JPEG/PNG/WEBP) or one PDF. Photos are downscaled client-side to a 2000px long edge before upload — a courtesy that cuts upload time and vision-token cost, never a control: every bound is re-checked server-side against magic bytes, a per-file cap, a total-size cap and a PDF page ceiling
- **Provider-agnostic by construction** — the wizard talks to a single `analyzeMenu()` interface and never learns which vendor answered. Adapters speak raw REST over the built-in `fetch`, so there is **no SDK dependency** and swapping providers is an env change (`AI_PROVIDER`, `AI_MODEL`), not a deployment of new packages
- **The model is never trusted.** Structured output is requested, but the answer is validated against a Zod schema server-side regardless. On a schema failure the concrete validation issues are fed back for exactly **one repair retry** before a typed error is raised — which is what makes it safe to run a small, cheap model here
- **Turkish price parsing is explicit**, because getting it wrong is expensive: `1.250,00 TL` → `1250`, never `1.25`. The dot/comma ambiguity is resolved by a locale-independent last-separator rule, and ranges ("100-150") are refused rather than guessed
- **It skips rather than invents.** A row the model cannot read is left out and reported in a warnings list, and the review step leads with that list — telling the owner plainly which items to add by hand. Rows it read but is unsure about are highlighted separately
- **Nothing is saved until approved** — the owner edits names and prices in place, unticks items or whole categories, and chooses append-or-replace when a menu already exists. The approved rows then enter the menu through the ordinary save path, so server-side normalisation, plan limits and the undo action all apply unchanged
- **Spend is capped in layers** — plan gate (Starter+), a per-actor hourly rate limit, and a per-restaurant daily quota on the Istanbul business day. A quota slot is reserved *before* the call so concurrent uploads can't slip through together, and it is refunded whenever an attempt returns nothing usable
- **Uploaded files are never stored** — they are analysis input and are dropped when the request ends

---

## PDF Menu *(Pro and Enterprise)*

For restaurants that already have a professionally designed menu, the public menu link can serve an uploaded PDF instead of the interactive templates.

- **Upload a PDF** from the Menu Design tab (PDF only, up to 10 MB); the file is validated by magic bytes server-side, not just the declared MIME type, and rejected past a generous page ceiling so a pathological file can't hurt the guest-side renderer
- **Display mode toggle** — switch the public view between **Interactive** and **PDF**; nothing goes live until an explicit **Save** (same draft/save pattern as the language and template settings), and uploading a PDF never silently changes what guests see
- **Mobile-friendly viewer** — guests open the same menu link and get a canvas viewer that renders **one page at a time as they scroll** (`IntersectionObserver`, pdf.js) rather than rasterising the whole document up front, so a long menu never freezes a low-RAM phone and the first page shows immediately. Placeholders reserve each page's height (no layout jump), a failed page retries on scroll-back, a floating "page N / total" indicator tracks position, and an "open in new tab" button is always available as an escape hatch
- **Plan-gated and downgrade-safe** — gated to Pro+ via the `pdfMenu` plan feature; the public endpoint forces the interactive view (and hides the PDF URL) for any restaurant below Pro, so a lapsed plan can't keep serving a PDF it can no longer manage
- Stored through the same object-store abstraction as menu images and served from the same static mount

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
- Transfer a whole table's bill to another table, or move **individual item rows** between tables (partial transfer)
- Merge and transfer are **status-safe**: combining rows keys on name + price + **status** + note, so a pending item is never folded into an already-served one (kitchen state and item notes are preserved)
- Close bill with payment method: Full payment / Equal split / Per-item — *recording payments requires Starter and above (see [Payment Tracking](#payment-tracking)); on the Free plan you can take orders but not close the bill*

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
  - Reprint button (gated by `kitchen.reprintTicket`) re-emits the same content with a `[REPRINT]` stamp; ticket text is stripped of control bytes so an order note can't inject printer commands
  - Offline queue: pending tickets persisted in localStorage and flushed when the printer reconnects
  - **Poison-ticket recovery**: a ticket that keeps failing at the USB level (bad bytes, a rejected command) is set aside after a few attempts so it can't wedge every ticket queued behind it — the KDS shows a "couldn't print" count so the kitchen can reprint that table by hand, while a device-level failure (unplugged cable) simply waits for the next retry and never quarantines a good ticket
- **Offline awareness**: a persistent banner appears the moment connectivity drops, so kitchen and waiter staff can tell a network outage from an app fault
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
- `GET /api/realtime/stream?channel=kitchen|tables|all` — restaurant-scoped, heartbeat every 25 s, re-checks token revocation on every beat
- Authenticated with a **single-use stream ticket**: the client `POST`s for a short-lived (30 s) opaque ticket over a normal Authorization-header request, then opens the stream with that ticket — the access token never rides in the EventSource URL (and so never lands in proxy/CDN logs or browser history)
- Event types: `order.created`, `order.itemAdded`, `order.itemPaid`, `order.itemCancelled`, `order.adjusted`, `order.statusChanged`, `order.sentToKitchen`, `table.merged`, `table.unmerged`, `table.transferred`, `table.closed`, `table.renamed`, `kitchen.ticketRendered`
- Single Node `EventEmitter` per process now; planned move to Redis Pub/Sub when we add a second backend instance
- EventSource on the client with capped exponential-backoff reconnect, a heartbeat watchdog, and refresh single-flight (a burst of reconnects shares one token refresh)

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
- **Atomic** — a single `POST /tables/:id/pay` endpoint books the `SalesRecord` and updates the table in one DB transaction; an optimistic-lock conflict rolls both back, so a retry can never half-settle a bill (the old two-call flow could double-collect or drop the revenue record on a network blip)
- **Server-validated totals** — the receipt lines and total are derived from the table's own billable rows; the client-supplied total is cross-checked and rejected if it doesn't add up, so the amount on the books can't be tampered with from the client. The manager-approval discount threshold is evaluated against the server-computed gross
- **Idempotent** — each pay action carries an idempotency key; a dropped response replayed on retry returns the original receipt instead of charging twice
- **Unique receipts** — receipt numbers (`YYYYMMDD-NNNN`) are issued under a per-restaurant advisory lock held to commit, backed by a unique index, so two concurrent payments can never share a number
- Used as input for the Analytics revenue charts

---

## Comp & Void

Available on Starter and above (reuses the `payments.applyDiscount` permission).

Item-level bill adjustments for the real-world "there's a hair in this dish" / wrong-item / customer-return cases — works even on an **already-served** line, which the plain pre-kitchen cancel cannot touch.

- **Two types** — *comp* (complimentary: food was made/served but not charged) and *void* (removed as an error or a customer return), tracked separately for reporting
- **Quantity-aware** with a **mandatory reason**; available both on the owner dashboard (table cart) and the waiter table-detail screen
- Removed lines stay **visible on the bill** (struck through, labelled as comped or returned) for guest + staff transparency, but are excluded from every total, payment picker and "amount owed"
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
| Comps | Value comped off bills, from the `OrderAdjustment` ledger |
| Voids | Value voided/returned off bills |
| Waste | Value of items that had reached the kitchen (`fromStatus` ≠ pending) and were never sold |
| Avg Order Value | Total revenue ÷ order count |

**Top Items table** — items ranked by quantity sold for the day.

**Top Adjustments table** — comped / voided / cancelled items ranked by value, with their type.

**PDF export** — generates an A4-format summary report via jsPDF + html2canvas.

---

## Day-End Report Email *(Pro and Enterprise)*

The same day-summary figures, delivered automatically. Owners opt in and pick a send hour; the report then arrives by email after each business day closes — no need to open the dashboard.

- **Opt-in, per restaurant** — off by default; a toggle plus an Istanbul send-hour selector (0–23) live on the Restaurant Settings page. The chosen hour defaults to 01:00 (day close + 1h buffer so late-night sales have settled)
- **Self-healing hourly job** — a cron tick computes the target day and the "already sent?" decision entirely from state (`istanbulDayStr(now-24h)` vs a `lastDayEndReportFor` stamp on the restaurant), so a missed run after a restart or downtime is picked up by the next tick with no backfill logic, and two ticks in the same day are idempotent (one email)
- **Contents** — revenue, receipt count, average bill, cash/card split, discount, comp/void totals and the top 5 products, formatted in Turkish (thousands `.`, decimals `,`) with the restaurant's currency symbol
- **Quiet on empty days** — a day with no sales is stamped and skipped (no email), which matters for the many inactive accounts; an SMTP failure is deliberately *not* stamped, so it retries on the next tick
- **Plan-gated** — gated to Pro+ via the `dayEndReportEmail` plan feature; the settings endpoint and the job both enforce it, so a Free/Starter (or downgraded) restaurant is never mailed
- Shares one aggregation service with the on-screen Day Summary, so the emailed numbers are byte-for-byte identical to the dashboard

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
1. User enters new email + current password in the change-email form
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
- **Rotating Refresh Tokens with reuse detection** — Stateless, HTTP-only refresh cookies (`Secure`, `SameSite=Strict`) that **rotate on every use** for both owner and staff sessions. The consumed token's JTI is recorded; a request bearing an already-consumed JTI means two parties hold the same cookie — a stolen-cookie replay — and force-logs-out the account. A short **grace window** absorbs the benign case (two browser tabs / a reload firing `/refresh` near-simultaneously) so it doesn't kick a legitimate user off every device. Auth is the JWT signature + Redis revocation, not a DB token column, so an owner can also stay signed in on more than one device.
- **Session invalidation on credential change** — `change-password` and `reset-password` flog out every live session (access + refresh) via a per-user timestamp; `change-password` hands the current device a fresh token so the user who just changed their password stays signed in there while every other device is dropped.
- **Redis Token Blacklist** — Revoked access tokens blacklisted by JTI; consumed refresh JTIs blacklisted on rotation; checked on every authenticated request and on each SSE heartbeat
- **No account enumeration** — Login returns one generic `InvalidCredentials` for both an unknown identifier and a wrong password, and the not-found path burns a cost-matched bcrypt so timing doesn't reveal existence either; resend-verification answers an unknown email exactly like a real send. The true reason is still recorded in the audit log.
- **Progressive CAPTCHA, outage-safe** — Cloudflare Turnstile triggered after repeated failed login attempts; verification fails safe on a provider outage (5 s timeout → `503` with `Retry-After`, never a hung request)
- **bcrypt Password Hashing** — Passwords hashed before storage; never stored in plaintext
- **Atomic signup** — User + Menu + Restaurant are created in a single transaction, so a mid-sequence failure can't leave a half-provisioned account
- **Rate Limiting, actor-keyed** — Per-IP progressive limiter on auth endpoints; global limiter across all API routes keyed by **verified actor id** (owner/staff) rather than IP, so a restaurant behind one NAT isn't throttled collectively, with a separate wider bucket for guest menu views
- **Dedicated manager approval PIN** — A manager can set an approval PIN separate from their login PIN; once set, only it green-lights over-threshold discounts/comps/voids, so a PIN entered in front of other staff never doubles as a login credential (the check is brute-force-locked either way)
- **Forgot Password** — Secure token-based reset flow:
  - 256-bit random token; SHA-256 hash stored in DB — raw token never persisted
  - 1-hour TTL; one-time use (cleared immediately on consumption)
  - Per-account 60-second resend cooldown; per-IP 5/hour rate limit
  - All active sessions invalidated on successful reset
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
