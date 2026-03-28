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
- Encodes a public URL pointing to the restaurant's live menu
- Downloadable as PNG from the dashboard

**Menu Templates (6)**
- **Classic** — Warm serif design, ideal for traditional restaurants
- **Minimal** — Clean monochromatic look for modern cafés
- **Modern** — Bold gradient with vibrant accents
- **Rustic** — Earthy, textured feel for farm-to-table or casual dining
- **Elegant** — Dark luxury with gold typography, ideal for fine dining
- **Neon** — Cyberpunk-inspired dark theme with glowing accents

Template access is gated by plan tier. Upgrading immediately unlocks the template for use.

**Live demo**
- A sample menu is always available at `/menu/sample` — no login or restaurant account required
- Uses the Rustic template with a complete Turkish restaurant dataset including item photos
- A floating template-switcher bar lets visitors preview all 6 templates on the same sample data

**QR Tabletop Card Designer**
- Each menu template has a matching QR tabletop card design
- Preview updates live as template is selected
- Export to PDF in three sizes: A5, A6, Square

---

## Table Management

- Create tables numbered sequentially
- Mark tables as occupied / free
- Track per-table order accumulation
- Merge a secondary table into a primary (combined bill view)
- Close bill with payment method: Full payment / Equal split / Per-item

---

## Payment Tracking

Available on Starter and above.

- Record sales when a table's bill is closed
- Payment methods: full, split, per-item
- Items and totals stored per transaction in `SalesRecord`
- Used as input for the Analytics revenue charts

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

---

## Internationalization

Every user-facing string in the dashboard, public menu, and error messages is localized in:
- **Turkish** (primary language)
- **English**

Language is set per restaurant and can be changed at any time from the dashboard settings. The language selection also affects the public guest menu.
