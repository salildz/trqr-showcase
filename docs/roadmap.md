# TRQR — Roadmap

This document outlines the public-facing development direction for TRQR.

---

## Recently Shipped

- **Comp / Void & Discount Approval Hardening** — The manager-approval gate that protects comps, voids and payment discounts is now production-grade. The threshold is a **per-restaurant setting** (owner-tunable 0–100%, default 5%, `0` = always require a PIN) instead of a hard-coded 5%, configurable from the Staff tab and governing both the comp/void and discount paths. The inline manager-PIN gained its own **brute-force lockout** (5 wrong PINs per `(restaurant, staff)` → 15-minute cool-off, refused before the bcrypt check, `429` with `Retry-After`, degrades open if Redis is down), and failed / locked-out attempts are now audit-logged. Comp/void requests are **idempotent** (a retry replays the original instead of double-comping), and the `OrderAdjustment` ledger now records **who approved** each over-threshold adjustment (manager id or restaurant override PIN).
- **Comp / Void / Waste & Operational Analytics** — Day Summary and Statistics surface comp (İkram), void (İptal-İade) and food-waste (Fire) totals from the `OrderAdjustment` ledger, plus staff- and kitchen-oriented breakdowns, all gated by analytics tier.
- **Copyable Public Menu Link** — The QR page surfaces the public menu URL up front in a read-only field with a one-tap copy button and copy/open shortcuts, so the owner can drop it straight into a social bio, WhatsApp or Google Business without opening the design panel.
- **Category-level "86" & Item-level Table Transfer** — Mark a whole menu category unavailable in one toggle, and move individual line items (not just whole tables) between tables during service.
- **Item-level Comp & Void (İkram / İptal-İade)** — Take an individual line off the bill — including an already-*served* dish, which the pre-kitchen cancel can't — as a *comp* (complimentary) or *void/return*, quantity-aware and with a mandatory reason. Surfaced on both the owner dashboard (table cart) and the waiter table-detail screen. Removed lines stay visible (struck through) for transparency but are excluded from every total / payment picker. Gated by `payments.applyDiscount`; a non-manager needs a manager PIN (verified server-side, "swipe a manager") to exceed the restaurant's configurable approval threshold (default 5%) — the same gate also governs payment-step discounts. Generalised the order-item state machine with terminal `comped` / `voided` statuses (a shared `NON_BILLABLE` set), and every adjustment — plus the existing pre-kitchen cancel — is written to a new append-only `OrderAdjustment` ledger that powers comp / void / food-waste totals in the Day Summary.
- **Staff Accounts & Roles** — Restaurant-managed sub-accounts with 6-digit PIN login, separate from the owner account. New "Personel" dashboard tab to create / edit / reset-PIN / deactivate; an "Advanced" panel fine-tunes the role's default permissions per user (e.g. turn off `payments.collect` for restaurants with a dedicated cashier). Roles: `manager` / `waiter` / `kitchen` / `cashier`. Each restaurant picks a table-assignment mode (`free` — every waiter sees every table; `assigned` — each table pinned to specific waiters with server-side enforcement). Backed by a new `StaffUser` model + its own JWT type and refresh cookie so an owner and a staff member can stay signed in on the same device.
- **Waiter App** at `/staff/waiter` — mobile-first, three-tab shell (Tables · Open Orders · Profile). Tables grid → detail → menu picker with quantity stepper + per-item notes ("no onion") → "Send to Kitchen" batches the pending items. Open Orders subscribes to the SSE channel, shows preparing + ready items grouped by table, vibrates the device when the kitchen marks something ready. "Servis Edildi" per-item with one tap. Payment flow: full / equal split / per-item, optional percentage discount, cash/card; records a `YYYYMMDD-NNNN` receipt and clears the table on success. Every action gated by the matching staff permission.
- **Kitchen Display System (KDS)** at `/staff/kitchen` — tablet-landscape card grid bucketed by minute of send. Each card shows a big table label, an elapsed `mm:ss` timer (ticks every 5 s), each line with optional italic note, strike-through for cancelled lines plus a warning banner. "Hazır" promotes every preparing line on the card; cards disappear automatically once everything is served or cancelled. Subscribes to the SSE `kitchen` channel for new tickets within ~400 ms of the waiter pressing Send.
- **Thermal Receipt Printing (WebUSB)** — Server-side ESC/POS rendering (`esc-pos-encoder`, cp857 for Turkish, 58 mm default) hands the kitchen tablet an opaque byte buffer. The KDS pairs with a USB thermal printer through the browser's WebUSB picker, auto-prints on every send-to-kitchen, and supports manual reprint with a `[REPRINT]` stamp. Offline queue in `localStorage` flushes on the next visibility change or pair, so a yanked cable doesn't lose a ticket. Curated vendor-ID filter covers Epson / Star / Bixolon / Citizen / Custom / SNBC / GLOBALPOS / Winbond clones; an "accept any" escape hatch handles off-brand printers. Browsers without WebUSB (Safari, Firefox) show an "unsupported" chip and the rest of the KDS keeps working.
- **Order Item State Machine** — Every line item now carries a stable UUID and a status (`pending → preparing → ready → served`, plus `cancelled` / `comped` / `voided` terminal states). Resolves the old array-index race when two waiters edited the same table simultaneously, and gives the KDS / waiter app concrete events to subscribe to. One-shot backfill migration stamped all existing items as `served` so legacy tabs never dumped onto the kitchen screen.
- **Server-Sent Events (SSE) Realtime Channel** — `GET /api/realtime/stream?channel=kitchen|tables|all` with per-restaurant scope, 25 s heartbeat, channel filtering and `X-Accel-Buffering: no` so Cloudflare / Nginx don't sit on small payloads. Per-event types: `order.created`, `order.itemAdded`, `order.statusChanged`, `order.itemCancelled`, `order.sentToKitchen`, `table.merged` / `.unmerged` / `.transferred` / `.closed` / `.renamed`, `kitchen.ticketRendered`. EventSource on the client with auto-reconnect, exponential backoff, visibility pause, and a 60 s heartbeat watchdog.
- **6 Menu Templates** — Classic, Minimal, Modern, Rustic, Elegant, Neon
- **QR Tabletop Card Designer** — Matching print cards per template, PDF export in A5/A6/Square
- **Print Menu with Item Images** — Thumbnails included in all 6 print templates
- **Table Management** — Per-table order tracking, table merge, payment splits
- **Analytics Dashboard** — Daily views, revenue, top items, hourly distribution
- **Immutable Audit Log** — Actor/IP/HTTP context/before-after diff on all mutations
- **Progressive CAPTCHA** — Cloudflare Turnstile triggered on repeated auth failures
- **Multilingual Support** — Full Turkish + English throughout
- **Plan Limit UX** — Specific, actionable errors on plan limit exceeded with one-click revert
- **JWT Token Blacklist** — Redis-backed immediate logout and access token revocation across all sessions
- **Email Verification** — Mandatory verification on signup; hard login block for unverified accounts; resend with DB-level 60s cooldown and 5/hour IP rate limit; email change before verification; bilingual TR/EN HTML emails with logo header via Google Workspace SMTP
- **Email Change Flow** — Authenticated users can change their registered email from the Restaurant Management page with password confirmation; SHA-256 token sent to the new address with 24 h TTL; 60 s resend cooldown; final uniqueness guard on confirmation; pending banner with cancel option in the dashboard
- **Live Sample Menu** — Fixed public URL `/menu/sample` serving a hardcoded Rustic-themed Turkish restaurant demo with category images; linked from the landing page and all QR-menu example pages; no login or DB lookup required
- **JSON-LD Structured Data** — Static `WebSite` and `Organization` schemas embedded in `index.html` for Wave 1 (static HTML) crawl; per-page `Restaurant` and `Menu` schemas injected by the SPA; resolves Google site-name display and improves rich-result eligibility
- **Percentage Discount** — Percentage-based discount applied at payment step alongside existing amount-based discounts; discount percent and computed amount persisted on the sales record
- **Receipt History** — Every completed payment stored as a sequentially numbered receipt (`YYYYMMDD-NNNN`); Receipts dashboard page with date/method filters, paginated list, itemised detail dialog, and per-receipt PDF export
- **Day Summary Report** — Aggregate report for any calendar date: order count, total/cash/card revenue, discount totals, average order value, top-selling items by quantity; exportable as thermal-style PDF
- **Forgot Password** — Secure token-based password reset flow: SHA-256 hashed token, 1-hour TTL, one-time use, per-account 60 s resend cooldown, per-IP rate limit (5/hour), all active sessions invalidated on reset; bilingual TR/EN reset email via SMTP
- **Receipt History for Free Plan** — Receipts and day-summary pages visible to all plans with a blur-lock overlay and upgrade CTA for users on the free tier
- **5-Day Free Trial** — Every new account starts on the Starter plan for 5 days with no credit card required. Trial expiry triggers a 3-day grace period (warning banner, no data loss). After the grace period, free-plan limits are enforced server-side: tables clamped to 5, categories to 5, item images to 12, and locked templates reset to Classic. Existing users registered within the trial window are backfilled automatically. The plan contact flow includes a mail-client fallback dialog with copyable email content for environments where mailto links don't open a mail app.
- **Custom Table Names** — Each table can be assigned a custom display name (e.g., "Garden", "VIP 1", "Terrace") via an inline rename dialog on the table card. Name is shown everywhere: card headers, order dialogs, merge/transfer pickers, and snackbar notifications. Clearing the name reverts to the default "Table N" label. Stored as a nullable `customName` column; reducing table count deletes from the highest-numbered end, preserving names for remaining tables.
- **Grace Period Feature Access** — During the 3-day grace period, all plan feature checks are bypassed so restaurants retain full operational access (payment recording, analytics, etc.) until enforcement day. The `loadPlan` middleware now sets `req.isInGracePeriod` from `gracePeriodEndsAt`; `requireFeature` and `requireAnalyticsTier` skip enforcement when this flag is set.
- **Grace Period Banner Improvements** — Dashboard warning banner now spans full page width regardless of per-page content max-width constraints. Template names in the impact detail line are translated to the user's language (e.g., "rustic" → "Rustik") with proper capitalisation.
- **Custom Menu URL (Slug)** — Pro and Enterprise restaurants can define a short, memorable slug for their public menu (e.g., `trqr.net/menu/yildiz-restoran`). Real-time availability check with debounce and reserved-word enforcement. Slug resolved server-side via `OR (restaurantPublicId, menuSlug)` lookup — both UUIDs and slugs work. QR code and "Go to Menu" link automatically reflect the active slug. Slug is cleared during grace period enforcement when the plan drops below Pro.
- **Multi-Currency Support** — Restaurants can set their display currency (TRY ₺, USD $, EUR €, GBP £) from the Restaurant Settings page. Currency symbol propagates to all price display points: table management, payment steps, receipt and day-summary PDFs, analytics totals, print menu export, and the live public guest menu. Stored as a `currency` column on the `Menu` model; updated via `PUT /api/menu/currency`. Frontend delivered through `CurrencyContext` (dashboard) and `MenuCurrencyContext` (public menu).
- **Watermark / Remove Branding** — "Powered by TRQR" watermark footer rendered at the bottom of every public menu. Styled per template (dark brown on Classic, gold on Elegant, neon green on Neon, etc.). Automatically hidden for Pro and Enterprise restaurants; never shown in preview mode. `removeBranding` flag resolved from the restaurant's plan server-side on the public menu endpoint.
- **Frontend Component Structure Refactor** — Dashboard tab components (`QrCode`, `PrintMenu`, `MenuManagement`, `Statistics`) moved to `features/dashboard/`; auth components (`SignIn`, `SignUp`, `ForgotPassword`) to `features/auth/`; shared utilities (`LanguageSwitcher`, `PageTransition`, `ProtectedRoute`, `RootRedirect`) to `common/`. `components/` root is now empty. All import paths updated; zero TypeScript errors after move.

---

## In Progress / Near-term

- **Multiple Kitchen Stations (Enterprise)** — Route line items to `grill` / `cold` / `bar` / `dessert` printer queues based on a per-item station tag, with each kitchen tablet subscribed to a single station via `?station=` query. Spec is finalised; tag column already lives on every order item.
- **Inventory Tracking** — Mark items as sold out with restock controls
- **Menu Item Variants** — Size / option groups (e.g., Small / Large, with price delta)
- **Customer Feedback** — Simple per-item or per-visit rating flow via QR scan

---

## Planned

- **Multi-location support** — One account managing multiple restaurant branches
- **Online Ordering** — Allow guests to submit orders directly from the menu
- **TRQR Print Agent (Enterprise)** — Small Node binary running on a Raspberry Pi / mini-PC in the restaurant LAN; takes ticket jobs over WebSocket and prints to USB / network printers. Adds resilience and offline buffering beyond what the browser-side WebUSB path delivers.
- **Reservation Module** — Embedded table reservation flow accessible via QR
- **Custom Domain for Menu** — Serve public menu at restaurant's own domain
- **WhatsApp / SMS Notifications** — Order confirmation and table-ready alerts

---

## Notes

- This roadmap reflects public product direction and does not commit to specific release dates.
- Feature availability may vary by plan tier at launch.
