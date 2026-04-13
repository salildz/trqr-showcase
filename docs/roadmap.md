# TRQR — Roadmap

This document outlines the public-facing development direction for TRQR.

---

## Recently Shipped

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

- **Inventory Tracking** — Mark items as sold out with restock controls
- **Menu Item Variants** — Size / option groups (e.g., Small / Large, with price delta)
- **Customer Feedback** — Simple per-item or per-visit rating flow via QR scan
- **Staff Roles** — Waiter / manager access levels separate from the owner account

---

## Planned

- **Multi-location support** — One account managing multiple restaurant branches
- **Online Ordering** — Allow guests to submit orders directly from the menu
- **Kitchen Display** — Real-time order feed for kitchen staff
- **Reservation Module** — Embedded table reservation flow accessible via QR
- **Custom Domain for Menu** — Serve public menu at restaurant's own domain
- **WhatsApp / SMS Notifications** — Order confirmation and table-ready alerts
- **RBAC Expansion** — Support / manager admin roles

---

## Notes

- This roadmap reflects public product direction and does not commit to specific release dates.
- Feature availability may vary by plan tier at launch.
