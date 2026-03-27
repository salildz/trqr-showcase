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
- **Email Verification** — Mandatory verification on signup; hard login block for unverified accounts; resend with DB-level 60s cooldown and 5/hour IP rate limit; email change before verification; bilingual TR/EN HTML emails via Google Workspace SMTP

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
- **JSON-LD Rich Results** — Structured data for `Restaurant` + `Menu` on public pages
- **RBAC Expansion** — Support / manager admin roles

---

## Notes

- This roadmap reflects public product direction and does not commit to specific release dates.
- Feature availability may vary by plan tier at launch.
