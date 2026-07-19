<div align="center">

<img src="assets/branding/logo.png" alt="TRQR Logo" width="200" />

### Restaurant Management System with Digital QR Menus

**Menu management, table tracking, payments, and analytics — all in one dashboard.**

[![Status](https://img.shields.io/badge/status-live-brightgreen?style=flat-square)](https://trqr.net)
[![Stack](https://img.shields.io/badge/stack-React%2019%20%2B%20Node.js%2022%20%2B%20PostgreSQL-blue?style=flat-square)](#tech-stack)
[![i18n](https://img.shields.io/badge/languages-TR%20%2F%20EN-orange?style=flat-square)](#internationalization)
[![License](https://img.shields.io/badge/source-private-lightgrey?style=flat-square)](#source-code)

---

[Live Demo](#demo) · [Features](#features) · [Tech Stack](#tech-stack) · [Architecture](#architecture) · [Plans](#subscription-plans) · [Screenshots](#screenshots)

</div>

---

## Overview

TRQR is a full-stack SaaS platform for restaurant management. At its core is a QR-based digital menu system — guests scan the table QR code, browse the menu instantly on their phone, and place orders with no app download required.

But TRQR goes well beyond menus. Restaurant owners get a complete operational dashboard: manage tables and live orders, track payments with split-bill support, monitor revenue and traffic analytics, design and print branded QR tabletop cards, and configure the entire system in Turkish or English — all in real time from a single interface.

Built with a strong emphasis on production security, plan-based access control, and a clean, fast user experience.

---

## Features

### For Restaurant Owners

| Feature | Description |
|---------|-------------|
| **Menu Builder** | Drag-and-drop category and item editor. Add names, prices, descriptions, and photos. The price field accepts both Turkish and English number formats (decimal comma or dot, thousands grouping) and echoes the parsed amount back live, so "2.500" is never silently stored as ₺2.50. Toggle a single item — or a whole category — to "out of stock" ("86") instantly; unavailable items are greyed out on the public menu and blocked from new orders. |
| **Menu Design** | Dedicated Menu Designer page with live preview (separate from Restaurant Settings). Six professionally designed templates, each with a matching QR tabletop card design. Template access gated by plan tier. |
| **QR Code & Tabletop Designer** | Unique QR code per restaurant with a built-in tabletop card designer. Preview all templates live, customise the card message, and export as PDF in A5, A6, or Square format. When a custom menu URL is set, the QR code encodes the slug URL; falling back to the UUID-based public URL otherwise. |
| **Custom Menu URL** | Pro and Enterprise restaurants can define a short, memorable slug for their public menu (e.g., `trqr.net/menu/yildiz-restoran`). Real-time availability checking with debounce. Slug is revoked automatically if the restaurant's plan falls below Pro after the grace period. |
| **Print Menu** | Export the full restaurant menu as a print-ready PDF. Choose template and paper size; preview renders live in the browser before export. |
| **PDF Menu** *(Pro and Enterprise)* | Upload a PDF and serve it on the public menu link instead of the interactive templates — ideal for restaurants that already have a professionally designed menu. Guests get a mobile-friendly viewer that renders the PDF page-by-page to a canvas as they scroll (so a long menu never freezes a low-RAM phone and the first page shows immediately), with a "page N / total" indicator and an open-in-new-tab escape hatch. Switching between interactive and PDF mode is an explicit save, and the public view automatically falls back to the interactive menu if the plan drops below Pro. |
| **Table Management** | Configure table count, assign custom names (e.g., "Garden", "VIP 1"), track per-table orders, merge tables, transfer a whole table or move individual items between tables, and close bills in one tap. Order state survives a merge/transfer (a pending item never silently becomes "served"). |
| **Staff & Kitchen System** | Owner-managed staff accounts with 6-digit PIN login (waiter / kitchen / cashier / manager roles, plus per-user permission overrides). A mobile waiter app for taking orders and collecting payment at the table, and a live Kitchen Display System (KDS) with thermal ticket printing. Available from the **Starter** plan; staff seat count scales with the tier. |
| **Payment Tracking** | Record full payments, equal splits, or per-item payments. Apply percentage discounts at checkout; all transactions stored with full line-item detail. Payments are **atomic** — the sale record and the table update commit in a single transaction, totals are recomputed server-side (the client total is never trusted), and each pay action carries an idempotency key so a dropped network response can't double-charge. Receipts are numbered uniquely per restaurant per day. |
| **Comp & Void (İkram / İptal-İade)** | Take an individual line — even an already-served dish — off the bill as a **comp** (complimentary, e.g. a quality complaint) or a **void/return** (wrong item), quantity-aware and with a mandatory reason. Removed lines stay visible on the bill (struck through) for transparency but are never charged. Gated by the discount permission; a non-manager needs a **manager's PIN** (a manager taps in, or an owner-set restaurant override PIN) to go past the restaurant's configurable approval threshold (default 5%) — no flat ceiling, and the PIN itself is brute-force-locked. Every adjustment is idempotent and lands in an append-only ledger that records who approved it. |
| **Receipt History** | Every completed payment stored as a numbered receipt (`YYYYMMDD-NNNN`). Filter by date/payment method, view itemised details, export individual receipts as PDF. |
| **Day Summary Report** | Aggregate report for any calendar date: orders, revenue, cash/card split, avg order value, discounts, **comps / voids / food-waste totals**, top items, and top adjustments by value. Exportable as PDF. |
| **Day-End Report Email** *(Pro and Enterprise)* | Opt-in daily email of the day-end summary — revenue, receipts, cash/card, comps/voids, top 5 products — sent automatically after each Istanbul business day closes, at an owner-chosen hour. A self-healing hourly job derives the target day and "already sent?" from state (so a missed run is caught next tick, and double-ticks never double-send); days with no sales are skipped, so inactive accounts aren't mailed. Shares one aggregation service with the on-screen Day Summary, so the numbers match exactly. |
| **Analytics Dashboard** | Daily views, revenue trends, top-selling items, and hourly traffic distribution — all with date range filtering (Today / This Week / This Month / Custom). |
| **Subscription Management** | In-dashboard plan management with live plan status, trial countdown, grace period warnings, and a direct upgrade path. |
| **Onboarding Wizard** | Step-by-step setup checklist for new restaurants: name, first item, menu preview, QR download. Dismissible once complete. |
| **Multi-Currency** | Set the display currency (TRY ₺, USD $, EUR €, GBP £) from Restaurant Settings. Currency symbol updates everywhere: table management, payment steps, receipts, analytics, print export, and the live public menu. |
| **Bilingual UI** | Full Turkish and English support throughout the entire dashboard, public menu, and email flows. |

### For Guests

- Instant menu access by scanning the table QR code — no app install
- Clean, responsive menu display on any device
- Category-based navigation with item photos, prices, and descriptions
- Template-matched visual design consistent with the restaurant's brand
- "Powered by TRQR" watermark shown on menus from Free and Starter plans; automatically removed for Pro and Enterprise restaurants

### For Staff (Waiters & Kitchen)

- **PIN login** at `/staff/login` — staff sign in with the restaurant's menu slug (or ID) + username + 6-digit PIN, independently of the owner account, on shared phones and tablets
- **Waiter app** (mobile-first) — table grid with live occupancy and an "awaiting payment" state, take orders with quantity steppers and per-item notes, send to kitchen, mark items served, and collect payment (full / equal split / per-item, with discounts)
- **Comp / Void at the table** — pull an individual line (including a served dish) off the bill as a comp (İkram) or void/return (İptal-İade) with a reason, right from the table detail screen; gated by permission, and anything past the restaurant's approval threshold (default 5%) prompts for a manager PIN inline
- **Kitchen Display System (KDS)** (tablet) — live order tickets grouped by table with elapsed timers, mark a whole ticket ready in one tap, and auto-print to a WebUSB thermal printer. The print queue survives a yanked cable (offline queue in `localStorage`) and recovers from a bad ticket: one that repeatedly fails at the USB level is set aside so it can't wedge the tickets behind it, with a "couldn't print" warning to reprint by hand
- **Offline-aware** — a persistent banner appears across the waiter and kitchen apps the moment the connection drops, so staff know it's the network, not the app; a failed request surfaces a clear "connection lost" message instead of failing silently
- **Open Orders** view — preparing and ready items grouped by table, with a buzz + toast the moment the kitchen marks something ready
- **Day-end report** — managers and cashiers can pull a shift/day summary (orders, revenue, cash/card split) from the staff app, scoped to their restaurant and computed on the Istanbul business day
- **Role- and permission-based** — every action gated by the staff member's effective permissions; assigned-table mode scopes a waiter to their own tables
- **Bilingual** — Turkish / English switch right in the staff login, waiter, and kitchen app bars

### For Platform Admins

- Full restaurant and user management
- Per-restaurant plan overrides and feature flag control
- Immutable audit log with actor, IP, HTTP context, and before/after diff
- Platform-wide analytics overview

---

## Screenshots

### Dashboard

| Menu Builder | Analytics |
|---|---|
| ![Menu Builder](assets/screenshots/dashboard-menu-builder.png) | ![Analytics](assets/screenshots/dashboard-analytics.png) |

| QR Code & Tabletop | Table Management |
|---|---|
| ![QR Code](assets/screenshots/qr-designer.png) | ![Tables](assets/screenshots/dashboard-tables.png) |

| Print Menu | Restaurant Settings |
|---|---|
| ![Print Menu](assets/screenshots/dashboard-print-export.png) | ![Restaurant Settings](assets/screenshots/dashboard-restaurant.png) |

| Menu Designer |
|---|
| ![Menu Designer](assets/screenshots/dashboard-menu-designer.png) |

### Public Menu — 6 Templates

| Classic | Minimal | Modern |
|---|---|---|
| ![Classic](assets/screenshots/menu-classic.png) | ![Minimal](assets/screenshots/menu-minimal.png) | ![Modern](assets/screenshots/menu-modern.png) |

| Rustic | Elegant | Neon |
|---|---|---|
| ![Rustic](assets/screenshots/menu-rustic.png) | ![Elegant](assets/screenshots/menu-elegant.png) | ![Neon](assets/screenshots/menu-neon.png) |

### Mobile

| Mobile Menu | Mobile Dashboard |
|---|---|
| ![Mobile Menu](assets/screenshots/mobile-menu.png) | ![Mobile Dashboard](assets/screenshots/mobile-dashboard.png) |

### Marketing Pages

| Landing Page | Pricing |
|---|---|
| ![Landing Page](assets/screenshots/landing-hero.png) | ![Pricing](assets/screenshots/landing-pricing.png) |

### Staff & Kitchen App

| Staff PIN Login | Waiter Tables |
|---|---|
| ![Staff Login](assets/screenshots/staff-login.png) | ![Waiter Tables](assets/screenshots/waiter-tables.png) |

| Open Orders (grouped by table) | Kitchen Display (KDS) |
|---|---|
| ![Open Orders](assets/screenshots/waiter-orders.png) | ![Kitchen Display](assets/screenshots/kitchen-display.png) |

| Collect Payment |
|---|
| ![Collect Payment](assets/screenshots/waiter-payment.png) |

### Authentication

| Login & Sign Up |
|---|
| ![Login](assets/screenshots/auth-login.png) |

---

## Tech Stack

### Frontend

| Layer | Technology |
|-------|-----------|
| Framework | React 19 + TypeScript 5.8 |
| Build Tool | Vite 6 |
| UI Components | Material UI (MUI) 7 |
| Styling | Emotion (CSS-in-JS) |
| Routing | React Router 7 |
| Animations | Framer Motion 12 |
| Drag & Drop | dnd-kit |
| i18n | i18next + react-i18next |
| PDF Export | jsPDF + html2canvas |
| SEO | React Helmet Async |
| CAPTCHA | Cloudflare Turnstile |
| HTTP Client | Axios |
| Testing | Vitest + Testing Library |

### Backend

| Layer | Technology |
|-------|-----------|
| Runtime | Node.js 22 |
| Framework | Express 5 |
| Database | PostgreSQL 16 |
| ORM | Sequelize 6 |
| Caching / Token Blacklist | Redis (ioredis) |
| Authentication | JWT + HTTP-only refresh tokens |
| Validation | Zod 4 |
| File Uploads | Multer 2 |
| Logging | Pino (structured JSON logs) |
| Scheduled Jobs | node-cron |
| Password Hashing | bcryptjs |
| Testing | Jest 30 + Supertest |

### Infrastructure

| Layer | Technology |
|-------|-----------|
| Containerization | Docker + Docker Compose |
| Reverse Proxy | Nginx |
| CDN / DDoS Protection | Cloudflare |
| Deployment | VPS with Nginx + SSL |
| Environment Config | `.env` file per environment |

---

## Architecture

```mermaid
graph TD
    A[Internet] --> B["Cloudflare\nCDN · DDoS Protection · Turnstile"]
    B --> C["Nginx\nReverse Proxy · TLS Termination"]
    C --> D["React SPA\nVite · TypeScript · MUI"]
    C --> E["Express API\nNode.js 22 · JWT · Zod · Pino"]
    E --> F[("PostgreSQL 16\nSequelize ORM")]
    E --> G[("Redis\ncache · token blacklist")]
    E --> H["File Storage\nmenu item images"]
```

See [`docs/architecture.md`](docs/architecture.md) for detailed component descriptions.

---

## Menu Templates

TRQR includes six professionally designed menu templates. Each template has a matching QR tabletop card design for print export.

| Template | Tier | Style |
|----------|------|-------|
| **Classic** | Free | Warm serif typography, cream/tan tones |
| **Minimal** | Free | Clean sans-serif, monochromatic |
| **Modern** | Starter | Bold gradient, vibrant accent color |
| **Rustic** | Starter | Earthy tones, dashed borders, handcrafted feel |
| **Elegant** | Pro | Dark luxury design, gold accents |
| **Neon** | Pro | Dark cyberpunk aesthetic, cyan/neon glow |

---

## Subscription Plans

> **Every new account starts on the Starter plan — free for the first 5 days.** No credit card required. After the trial, accounts automatically move to the Free plan unless upgraded. A 3-day grace period follows expiry before any data is trimmed.

| | **Free** | **Starter** | **Pro** | **Enterprise** |
|---|---|---|---|---|
| **Price** | ₺0 | ~~₺1,999~~ ₺999 / mo | ~~₺2,999~~ ₺1,499 / mo | Custom |
| **Trial** | — | **5 days free** | — | — |
| Tables | 5 | 15 | 40 | 100 |
| Categories | 5 | 15 | 30 | 50 |
| Items per Category | 15 | 30 | 50 | 100 |
| Items with Photos | 12 | 60 | 200 | 1,000 |
| Templates | Classic, Minimal | + Modern, Rustic | + Elegant, Neon | All |
| Analytics | Basic | Standard | Advanced | Premium |
| Payment Tracking | — | ✓ | ✓ | ✓ |
| Staff Accounts | — | 3 | 10 | 50 |
| Waiter App + Kitchen Display (KDS) | — | ✓ | ✓ | ✓ |
| Data Export | — | — | ✓ | ✓ |
| Remove Branding | — | — | ✓ | ✓ |
| Priority Support | — | — | ✓ | ✓ |
| Custom Menu URL | — | — | ✓ | ✓ |
| PDF Menu | — | — | ✓ | ✓ |
| Day-End Report Email | — | — | ✓ | ✓ |
| Custom Domain | — | — | — | ✓ |

Plan limits are enforced **server-side** — no client-side bypassing.

---

## Security Highlights

TRQR is built with a production security mindset throughout.

- **No account enumeration** — Login returns a single generic error for both an unknown identifier and a wrong password (the miss path burns a cost-matched bcrypt so response time doesn't leak account existence either), and resend-verification answers an unknown address exactly like a real one. The full reason still lands in the audit log for investigation.
- **Progressive CAPTCHA, outage-safe** — Cloudflare Turnstile is triggered after repeated failed login attempts (hard block past the threshold); verification fails safe on a provider outage — a 5 s timeout returns `503` with `Retry-After` rather than hanging the request or wrongly rejecting a human.
- **Rotating refresh tokens with reuse detection** — Short-lived access tokens; HTTP-only cookie refresh tokens that **rotate on every use**. A replayed (already-consumed) refresh token is treated as a stolen-cookie signal and force-logs-out the account, with a short grace window so a benign multi-tab/reload race doesn't trip it. Password change/reset invalidate every other live session. The legacy plaintext refresh column has been dropped — the JWT signature plus Redis revocation are the sole source of truth.
- **Single-use SSE stream tickets** — The realtime (Server-Sent Events) channel is authenticated with a short-lived, single-use ticket instead of the access token, so the token never lands in a URL, proxy log, or browser history.
- **Manager-PIN approval, brute-force locked** — Over-threshold discounts/comps/voids require a manager PIN (or an owner-set override PIN); the check is rate-limited and locks out after repeated failures. A manager can set a **dedicated approval PIN** distinct from their login PIN, so the code entered in front of other staff to approve an adjustment never doubles as a login credential.
- **Thermal-printer command hardening** — Kitchen-ticket text (item names, notes, table/waiter labels) is stripped of control bytes before it reaches the ESC/POS encoder, so a crafted order note can't smuggle printer commands (drawer-kick, cutter) into the byte stream.
- **Zod Validation** — All incoming API data validated against strict schemas before reaching business logic; JSONB row payloads are whitelisted (no passthrough) so client junk can't accumulate in the database.
- **XSS Sanitization** — Recursive sanitization on every request body, query, and params
- **Helmet + Nginx headers** — Full HTTP security header set (CSP, HSTS, X-Frame-Options, nosniff, referrer policy), with the edge headers centralised so a per-location cache rule can't silently drop them.
- **Rate Limiting, per-actor** — Tiered limiters per endpoint category (auth, uploads, global). Authenticated traffic is keyed by **verified actor id** rather than IP, so a whole restaurant behind one NAT isn't throttled as a single client; guest menu views get their own wider bucket.
- **HPP Protection** — HTTP Parameter Pollution prevention
- **Request Correlation** — Every request gets a UUID at ingress; propagated to audit logs
- **Immutable Audit Log** — Every meaningful server action is logged with actor, IP, HTTP context, resource, and before/after diff; retained on a rolling window by a scheduled cleanup job
- **Plan Enforcement** — Server-side middleware validates feature access and resource limits on every mutation
- **Dependency hygiene** — Both the client and server dependency trees audit clean (0 known vulnerabilities), kept current as part of routine maintenance.

---

## Internationalization

Full bilingual support in **Turkish** and **English** — including dashboard, error messages, email flows, and the public menu page.

Language is configurable per restaurant and switchable in the dashboard at any time.

---

## Demo

🌐 **Live Product:** [https://trqr.net](https://trqr.net)

---

## Source Code

**This repository is a public showcase only. The application source code is private.**

TRQR is a commercial product. The source code — including the frontend, backend, database migrations, infrastructure configuration, and all business logic — is maintained in a private repository and is not available for public access or contribution.

This showcase exists to document the product's architecture, features, and technical decisions for portfolio and reference purposes.

For licensing inquiries, collaboration, or white-label arrangements, please reach out via the contact section below.

---

## About

Built and maintained by **Salih Yıldız**.

- **Portfolio:** [yildizsalih.com](https://yildizsalih.com)
- **GitHub:** [@salildz](https://github.com/salildz)

---

<div align="center">

*TRQR — Bringing restaurant menus into the digital era.*

</div>
