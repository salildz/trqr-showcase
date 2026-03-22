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

## Internationalization

Every user-facing string in the dashboard, public menu, and error messages is localized in:
- **Turkish** (primary language)
- **English**

Language is set per restaurant and can be changed at any time from the dashboard settings. The language selection also affects the public guest menu.
