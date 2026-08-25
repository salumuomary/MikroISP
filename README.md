# MikroISP Manager

A PHP web application for managing MikroTik RouterOS hotspot / User Manager deployments.
Supports multiple routers, voucher lifecycle management, PoS agents with commissions, EPOS thermal printing, and PDF voucher export with themeable templates.

---

## Table of Contents

1. [Stack &amp; Requirements](#1-stack--requirements)
2. [Project Structure](#2-project-structure)
3. [Pages](#3-pages)
4. [API Endpoints](#4-api-endpoints)
5. [Data Storage](#5-data-storage)
6. [Voucher Metadata Schema](#6-voucher-metadata-schema)
7. [EPOS Print System](#7-epos-print-system)
8. [PDF Export &amp; Theme System](#8-pdf-export--theme-system)
9. [Adding a New Theme](#9-adding-a-new-theme)
10. [UI Design System](#10-ui-design-system)
11. [Navbar Structure](#11-navbar-structure)
12. [Changelog](#12-changelog)

---

## 1. Stack & Requirements

| Layer         | Technology                                          |
| ------------- | --------------------------------------------------- |
| Server        | PHP 8.x (via Laragon on Windows)                    |
| Router API    | MikroTik RouterOS API (raw socket, port 8728)       |
| Database      | MySQL (`mikroisp_system`) + RouterOS User Manager |
| Frontend      | Vanilla JS + custom CSS (no frameworks)             |
| Font          | JetBrains Mono (Google Fonts)                       |
| PDF / PNG     | html2canvas 1.4.1 (CDN, client-side)                |
| Composer deps | `mike42/escpos-php`, `tecnickcom/tcpdf`         |

### Composer packages

```bash
composer install
```

`composer.json`:

```json
{
    "require": {
        "mike42/escpos-php": "^4.0",
        "tecnickcom/tcpdf": "^6.11"
    }
}
```

> **Note:** The native raw-socket RouterOS APIs do not depend on the composer packages. Voucher rendering/export remains mostly client-side.

---

## 2. Project Structure

```
mikroisp_system/
│
├── README.md                          ← this file
│
├── ── Root pages ──────────────────────────────────────────────────
├── index.php                          redirect → login or dashboard
├── login.php                          session authentication
├── logout.php
├── dashboard.php                      system overview & live stats
│
├── ── Voucher pages ───────────────────────────────────────────────
├── vouchers.php                       full voucher list
├── create_vouchers.php                single / bulk voucher generator
├── usage.php                          per-voucher usage / uptime logs
├── deleted_vouchers.php               archive of soft-deleted vouchers
├── voucher_export.php                 export & print hub (CSV / Excel / EPOS / PDF)
├── voucher_preview.php                popup — renders vouchers via selected theme
│
├── ── Agent / Accounting pages ────────────────────────────────────
├── pos_agents.php                     PoS agent CRUD
├── agent_commissions.php              commission ledger per agent
├── accounting.php                     general ledger
│
├── ── Profile / Router pages ──────────────────────────────────────
├── allrouters.php                     router list
├── addrouter.php                      add / edit router
├── manage_profiles.php                RouterOS User Manager profiles
├── manage_limitations.php             bandwidth / quota limitations
├── profile_limitations.php            profile ↔ limitation mapping
│
├── ── Advanced / System pages ─────────────────────────────────────
├── hotspot_settings.php
├── radius_settings.php
├── system_settings.php
├── account_settings.php
├── file_manager.php
├── file_editor.php
├── terminal.php
│
├── ── Includes (backend logic) ────────────────────────────────────
├── includes/
│   ├── navbar.php                     reusable nav component (desktop + mobile)
│   ├── auth.php                       legacy auth helper
│   ├── auth_guard.php                 current session/permission guard
│   ├── db.php                         PDO connection helpers
│   ├── db_helpers.php                 app data helpers (routers, agents, printers, settings)
│   ├── ros_api.php                    shared raw-socket RouterOS helpers
│   ├── router_usage_cache.php         shared voucher snapshot cache
│   ├── router_dashboard_cache.php     shared dashboard snapshot cache
│   │
│   ├── ── API files ───────────────────────────────────────────────
│   ├── usage_api.php                  GET cached voucher list + POST delete/update_comment
│   ├── create_vouchers_api.php        GET profiles/agents + POST create vouchers
│   ├── agents_api.php                 CRUD for MySQL-backed agents
│   ├── voucher_export_api.php         GET cached export list + POST update_print / export_excel
│   ├── print_api.php                  GET printers + POST print (ESC/POS, network/USB)
│   ├── pdf_theme_api.php              GET themes/defaults + POST set_default/save_defaults
│   ├── user_history_api.php           voucher activity history + renew/update_comment
│   ├── hs_api.php                     hotspot settings API
│   ├── um_api.php                     user manager API
│   ├── dash_connect.php               dashboard snapshot load
│   ├── dash_live.php                  dashboard poll endpoint
│   └── dash_users.php                 dashboard user stats
│
├── mikroisp_system.sql                current MySQL schema dump
└── password_reset_tokens.sql          password reset table dump
│
└── ── Assets ──────────────────────────────────────────────────────
    assets/
    ├── css/
    │   ├── main.css
    │   └── inter.css
    ├── fonts/inter/                   self-hosted Inter woff2 files
    ├── logo/
    │   ├── logo.png
    │   └── logo-h.png
    ├── images/
    │   ├── icons/                     SVG icons + MikroTik product icons
    │   └── rb_images/                 RouterBoard product images (.webp)
    ├── epos_print/
    │   ├── default_print_details.json EPOS printer & voucher config
    │   └── 1-8.png                    icon assets for thermal receipts
    └── pdf_exports_settings/
        ├── vouchers_defaults.json     global voucher appearance config
        ├── preview.php                legacy single-theme preview helper
        ├── prev.php                   legacy neon-theme preview helper
        ├── basic-theme/
        │   ├── theme.json             theme metadata
        │   ├── preview.png            thumbnail shown in Theme Manager
        │   ├── v-theme.html           voucher DOM template
        │   ├── v-theme.css            voucher styles
        │   └── v-theme.js             renderVoucher(config, voucherDetails)
        └── neon-theme/
            ├── theme.json
            ├── preview.png
            ├── v-theme.html
            ├── v-theme.css
            └── v-theme.js
```

---

## 3. Pages

### `vouchers.php`

Full voucher list pulled live from RouterOS User Manager.
Filter tabs: All · Online · Printed · Unprinted · SMS Sent · Trial.
Actions per row: view history, edit comment, delete.

### `create_vouchers.php`

Generate vouchers in **single** or **bulk** mode.
Supports user types (`normal-user`, `longterm-user`, `trial-user`), agent assignment, payment gateway, custom prefix/length.

### `voucher_export.php`

Export & print hub. Three filter tabs:

| Tab       | Filters available                               |
| --------- | ----------------------------------------------- |
| By Date   | Date from/to, user type, status, price range    |
| By Agent  | Agent, user type, date range, status, min price |
| By Status | Status, user type, agent, date range, min price |

Toolbar actions:

- **CSV** — client-side download, UTF-8 BOM
- **Excel** — client-side SpreadsheetML `.xls`
- **PDF Export** — opens PDF modal → preview popup via `voucher_preview.php`
- **Print All Filtered** — sends to EPOS thermal printer via `print_api.php`

Selection: checkboxes per row + "Select All on page". Floating selection bar shows count + bulk-print button.

### `voucher_preview.php`

Popup page that renders vouchers through a theme's `v-theme.html/css/js`.
Uses an off-screen **scratch pad** strategy: one voucher is rendered at a time using the theme's own `renderVoucher()`, the `.voucher-container` is deep-cloned with IDs stripped, then moved to the visible print stage. This ensures `document.getElementById()` works correctly even for 100+ vouchers.

Modes:

- `?single=1` — one voucher + **Download PNG** (html2canvas, 3× scale)
- default — all vouchers, browser Print / Save PDF

### `pos_agents.php`

Add/edit/deactivate PoS agents. Agents are stored in MySQL and referenced by integer key inside each voucher's RouterOS comment JSON.

### `deleted_vouchers.php`

Displays vouchers archived in MySQL when a voucher is deleted from `usage.php`.

---

## 4. API Endpoints

All APIs return `Content-Type: application/json`.
All responses include `"ok": true|false`. On failure, `"error"` contains a message.

---

### `includes/usage_api.php`

| Method   | Params                                            | Description                                                                 |
| -------- | ------------------------------------------------- | --------------------------------------------------------------------------- |
| `GET`  | `?router_id=X`                                  | Read voucher list from shared MySQL cache; refresh from RouterOS when stale |
| `POST` | `action=delete`, `.id`                        | Remove voucher from User Manager                                            |
| `POST` | `action=update_comment`, `.id`, `comment{}` | Write updated JSON back to RouterOS comment and sync cache                  |

---

### `includes/create_vouchers_api.php`

| Method   | Params                                                                            | Description                       |
| -------- | --------------------------------------------------------------------------------- | --------------------------------- |
| `GET`  | `?router_id=X`                                                                  | Get available profiles and agents |
| `POST` | `action=create`, `mode=single\|bulk`, `profile`, `userType`, `agent`, … | Create vouchers on RouterOS       |

---

### `includes/voucher_export_api.php`

| Method   | Params                                            | Description                                                                          |
| -------- | ------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `GET`  | `?router_id=X`                                  | Read export voucher list from shared MySQL cache with flattened fields for filtering |
| `POST` | `action=update_print`, `.id`, `comment{}`   | Update single voucher print status in RouterOS comment and cache                     |
| `POST` | `action=bulk_update_print`, `updates[]`       | Batch update print status in RouterOS comment and cache                              |
| `POST` | `action=export_excel`, `rows[]`, `filename` | Stream SpreadsheetML`.xls` file                                                    |

---

### `includes/print_api.php`

| Method   | Params                                                                      | Description                                                                                                                                        |
| -------- | --------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `GET`  | —                                                                          | Return printer list from MySQL + EPOS config                                                                                                       |
| `POST` | `action=print`, `printer_id`, `router_id`, `vouchers[]`, `copies` | Generate ESC/POS bytes, send to network printer (TCP socket) or USB printer (`copy /b` to LPT1), then update RouterOS comment `print.*` fields |

**ESC/POS features:** business name (double-size), voucher code (double-width bold), profile + price, message, footer, date/time, contact, cut command.

---

### `includes/pdf_theme_api.php`

| Method   | Params                                   | Description                                                                                                        |
| -------- | ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `GET`  | `?action=themes`                       | List all theme folders, each with name/description/version/author,`preview_url`, `has_preview`, `is_default` |
| `GET`  | `?action=defaults`                     | Read`vouchers_defaults.json`                                                                                     |
| `POST` | `action=set_default`, `folder`       | Update`default_theme` in `vouchers_defaults.json`                                                              |
| `POST` | `action=save_defaults`, `defaults{}` | Overwrite entire`vouchers_defaults.json`                                                                         |

---

### `includes/agents_api.php`

| Method   | Params                     | Description                 |
| -------- | -------------------------- | --------------------------- |
| `GET`  | —                         | List all agents             |
| `POST` | `action=add\|edit\|delete` | CRUD on MySQL-backed agents |

---

## 5. Data Storage

Current storage is split into three groups:

### MySQL (`mikroisp_system`)

Main application entities now live in MySQL:

- routers
- agents
- printers
- PDF export defaults
- EPOS print settings
- deleted voucher archive
- password reset tokens
- shared router caches:
  - `router_cache_state`
  - `router_usage_cache`
  - `router_dashboard_cache`

### RouterOS / User Manager

Live hotspot and voucher state still comes from MikroTik:

- User Manager users
- User Manager profiles and user-profile links
- sessions
- hotspot active users
- PPP active users

Voucher metadata is still stored in each RouterOS user's `comment` field as JSON.

### Remaining JSON files

Some legacy/partial runtime files still exist:

- `assets/epos_print/default_print_details.json`
- `assets/pdf_exports_settings/vouchers_defaults.json`
- theme `theme.json` files
- `agentscollections.json` and related accounting JSON files in older accounting flows

The core voucher list, export, dashboard, routers, agents, printers, and settings are no longer JSON-backed.

### Background cache refresh job

Use the CLI job to refresh router caches on a schedule instead of waiting for page requests:

- Script: `scripts/refresh_router_caches.php`
- Refresh all routers: `php scripts/refresh_router_caches.php`
- Force refresh now: `php scripts/refresh_router_caches.php --force`
- Refresh one owner: `php scripts/refresh_router_caches.php --owner=1`
- Refresh one router: `php scripts/refresh_router_caches.php --router=router_abc123`

Recommended schedule:

- Dashboard-heavy environments: every 1 minute
- Lower load: every 5 minutes

Windows Task Scheduler action example:

- Program: `C:\laragon\bin\php\php-8.3.14-Win32-vs16-x64\php.exe`
- Arguments: `C:\laragon\www\mikroisp_system\scripts\refresh_router_caches.php`
- Start in: `C:\laragon\www\mikroisp_system`

Linux cron example:

- `* * * * * /usr/bin/php /var/www/mikroisp_system/scripts/refresh_router_caches.php >> /var/log/mikroisp/cache-refresh.log 2>&1`

### `assets/epos_print/default_print_details.json`

```json
{
  "business": { "name": "WiFi Kitonga", "phone": "0786749426", "currency": "TZS" },
  "printer":  { "paper_width": 32, "copies": 1 },
  "voucher":  { "show_logo": true, "message": "...", "show_price": true, "show_duration": true, "show_datetime": true },
  "contact":  { "label": "Kwa maelezo zaidi, Piga:" },
  "formatting": { "title_size": [2,2], "code_size": [2,1], "normal_size": [1,1], "align": "center" }
}
```

### `assets/pdf_exports_settings/vouchers_defaults.json`

```json
{
  "business": { "default_theme": "/basic-theme/", "name": "WiFi Kitonga" },
  "voucher": {
    "show_logo": true,
    "logo_path": "../logo/logo.png",
    "show_little_icon": true,
    "little_icon": "../epos_print/3.png",
    "message": "***--Tunza vocha yako mpaka muda uishe--***",
    "show_price": true,
    "show_duration": true
  },
  "contact": { "label": "Kwa maelezo zaidi:", "phone": "Piga: 0786749426" }
}
```

---

## 6. Voucher Metadata Schema

Each RouterOS User Manager user's **comment** field stores a JSON object:

```json
{
  "name": "normal-user",
  "userType": "normal-user",
  "agent": 11,
  "generation": {
    "date": "2026-04-06",
    "time": "14:51:24",
    "type": "printed-copy"
  },
  "print": {
    "printed": true,
    "count": 1,
    "date": "2026-04-07",
    "time": "09:30"
  },
  "sms": {
    "sent": false,
    "date": null,
    "time": null
  },
  "bulkSms": {
    "sent": false,
    "date": null,
    "time": null
  },
  "client": {
    "reference": null,
    "mac": null,
    "phone": null
  },
  "financial": {
    "totalResales": 1000.0,
    "paymentGatewayName": null
  }
}
```

| Field                      | Type                                                   | Description                                    |
| -------------------------- | ------------------------------------------------------ | ---------------------------------------------- |
| `name`                   | string                                                 | Display name                                   |
| `userType`               | `normal-user` \| `longterm-user` \| `trial-user` | Voucher category                               |
| `agent`                  | integer                                                | Key into the MySQL`agents` table / agent map |
| `generation.type`        | `printed-copy` \| `sms-sent` \| …                 | How the voucher was distributed                |
| `print.count`            | integer                                                | Total times printed (increments on each print) |
| `financial.totalResales` | float                                                  | Sale price in local currency (TZS)             |

---

## 7. EPOS Print System

### Flow

```
voucher_export.php
  └─ POST includes/print_api.php?action=print
        ├─ generate ESC/POS bytes per voucher (PHP)
        ├─ send bytes → TCP socket (network printer)  OR  copy /b to LPT1 (USB)
        └─ update RouterOS comment: print.printed=true, print.count++, print.date/time
```

### Printer types

| `type` in `printers.json` | Send method                                          |
| ----------------------------- | ---------------------------------------------------- |
| `network-printer`           | `fsockopen(ip, port)` → raw TCP write             |
| `usb-printer`               | Write`.prn` temp file → `copy /b file.prn LPT1` |

### ESC/POS layout (per voucher)

```
[Business Name — double height+width, centered]
================================
[Voucher Code — double width, bold, centered]
[Profile | Price TZS]
================================
[Message text]
[Footer message]
[Date Time    Contact phone]
================================
[3× line feed]
[Partial cut]
```

---

## 8. PDF Export & Theme System

### Flow

```
voucher_export.php
  └─ opens voucher_preview.php?theme=<folder>[&single=1]  (popup)
        ├─ PHP: loads v-theme.html, v-theme.css, v-theme.js, vouchers_defaults.json
        ├─ JS:  reads voucher data from localStorage key "vex_preview_vouchers"
        ├─ For each voucher:
        │     1. Inject THEME_HTML into off-screen scratch pad (#vp-scratch)
        │     2. Call theme's renderVoucher(defaults, voucherDetails)  ← IDs resolved correctly
        │     3. Deep-clone .voucher-container
        │     4. Strip all IDs from clone
        │     5. Clear scratch pad
        │     6. Append clone to #vpStage
        └─ User clicks Print / Save PDF  →  browser print dialog
           OR  Download PNG  →  html2canvas(target, {scale:3})
```

### Theme Manager (in `voucher_export.php`)

Opened via **PDF Export → Browse Themes**.

- Displays `preview.png` from each theme folder as a card thumbnail
- **★ Set Default** → `POST pdf_theme_api.php {action:"set_default", folder:"neon-theme"}`
- **Preview** → opens `voucher_preview.php?theme=neon-theme&single=1` with a sample voucher

### Voucher Settings Editor (in `voucher_export.php`)

Opened via **PDF Export → Edit Settings**.Edits all fields in `vouchers_defaults.json`:

- Business name
- Logo path, Little icon path (show/hide toggles)
- Message text
- Show price / show duration toggles
- Contact label + phone

Saved via `POST pdf_theme_api.php {action:"save_defaults", defaults:{…}}`.

---

## 9. Adding a New Theme

1. Create a folder: `assets/pdf_exports_settings/my-theme/`
2. Add `theme.json`:

```json
{
  "Theme": {
    "name": "My Theme",
    "description": "Short description.",
    "version": "1.0.0",
    "author": "Your Name"
  }
}
```

3. Add `preview.png` — a screenshot thumbnail of the voucher (any size; displayed at 100% width in a 240px card).
4. Add `v-theme.html` — the voucher DOM template. Use these element IDs as placeholders:

| ID                | Populated with                    |
| ----------------- | --------------------------------- |
| `biz-name`      | Business name                     |
| `v-code`        | Voucher username / code           |
| `v-duration`    | Profile (rendered as "Muda: …")  |
| `v-price`       | Price (rendered as "Bei: …")     |
| `contact-label` | Contact label text                |
| `contact-phone` | Contact phone                     |
| `v-message`     | Footer message                    |
| `main-logo`     | `<img>` — logo src + show/hide |
| `little-icon`   | `<img>` — icon src + show/hide |

5. Add `v-theme.css` — styles scoped to `.voucher-container`.
6. Add `v-theme.js` — must define:

```javascript
function renderVoucher(config, voucherDetails) {
    // config     = vouchers_defaults.json contents
    // voucherDetails = { code, duration, price }
    document.getElementById('biz-name').innerText  = config.business.name;
    document.getElementById('v-code').innerText    = voucherDetails.code || '';
    // … populate all IDs …
}
```

> The preview system automatically discovers the new folder and includes it in the Theme Manager.

---

## 10. UI Design System

All pages share the same inline CSS design tokens.

### Color Palette

| Token            | Value       | Usage                                 |
| ---------------- | ----------- | ------------------------------------- |
| `--primary`    | `#009345` | Buttons, active states, green accents |
| `--surface`    | `#181b1f` | Cards, table headers, navbar          |
| `--bg`         | `#0f0f12` | Page background                       |
| `--text`       | `#fcfdfd` | Primary text                          |
| `--text-muted` | `#8892b0` | Secondary text, labels                |
| `--accent`     | `#ad917a` | Tan accent (avatar)                   |
| `--accent2`    | `#fd7a2a` | Orange accent                         |
| `--danger`     | `#d32f2f` | Destructive actions                   |
| `--info`       | `#2196f3` | Info badges                           |
| `--warn`       | `#f59e0b` | Warning states                        |

### Typography

- Font: **JetBrains Mono** (Google Fonts, monospace)
- Page title: `1.6rem / 700`
- Body: `0.76–0.82rem`
- Labels/badges: `0.65–0.72rem`

### Component Classes

| Class                                                        | Description                                                                                                       |
| ------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| `.page-wrap`                                               | `max-width: 1600px`, centered, padded                                                                           |
| `.btn`                                                     | Base button — flex,`0.76rem`, `8px` border-radius                                                            |
| `.btn-primary`                                             | Green`#009345` fill                                                                                             |
| `.btn-ghost`                                               | Transparent with subtle background                                                                                |
| `.btn-orange` `.btn-blue` `.btn-danger` `.btn-green` | Tinted variant buttons                                                                                            |
| `.btn-sm`                                                  | Smaller padding/font variant                                                                                      |
| `.badge`                                                   | Pill badge —`.b-green` `.b-orange` `.b-blue` `.b-red` `.b-warn` `.b-purple` `.b-cyan` `.b-muted` |
| `.stat-card`                                               | Dark surface card with stat label + value                                                                         |
| `.filter-tabs` / `.ftab`                                 | Tab strip,`.active` → green fill                                                                               |
| `.tbl-wrap`                                                | Overflow-x scroll container                                                                                       |
| `table`                                                    | Collapsed, sticky`thead`, hover rows                                                                            |
| `.modal-overlay` / `.modal`                              | Fixed centered overlay + white-surface card                                                                       |
| `.toast`                                                   | Fixed bottom-right notification —`.success` `.error` `.info`                                               |
| `.spinner`                                                 | CSS rotation animation                                                                                            |
| `.pagination` / `.pag-btn`                               | Page navigation row                                                                                               |

### Responsive

- Breakpoint: `≤ 700px` — single-column stats, full-width search, stacked toolbar
- Navbar breakpoint: `≤ 1300px` — desktop menu hidden, mobile drawer activated

---

## 11. Navbar Structure

File: `includes/navbar.php`

```
Mikrotik HS Manager (brand)
│
├── Dashboard
├── Routers ▾
│   ├── All Routers
│   └── Add New Router
├── Vouchers ▾
│   ├── Voucher List
│   ├── Create Vouchers
│   ├── Usage Logs
│   ├── PoS Agents
│   └── Export & Print          ← voucher_export.php
├── Accounting ▾
│   ├── General Ledger
│   ├── Deleted Vouchers
│   └── Commissions
├── Profiles ▾
│   ├── Bundle Profiles
│   ├── Limitations
│   └── Profile-Limits Map
├── Advanced ▾
│   ├── Radius & UM
│   ├── Hotspot Settings
│   ├── File Manager
│   └── Terminal
└── System
```

Mobile: slide-in drawer with collapsible sections (`mobToggle()`).

---

## 12. Changelog

### Current architecture

- Core app entities are now MySQL-backed, not JSON-backed.
- Voucher list, export list, and dashboard reads use shared MySQL cache tables before re-pulling RouterOS.
- Dashboard interface rates no longer rely on request-time `sleep(1)` sampling; rates are derived from cached interface snapshots.
- Voucher metadata still lives in RouterOS User Manager `comment` JSON.

### 2026-04-07 (update 3)

- **`dashboard.php`** — Enhanced dashboard statistics
  - Added **User Activity** section with two new cards: "Users Today" and "Users This Week" (unique sessions since midnight / since Monday)
  - Each user-activity card shows session count + income generated by those users as a sub-stat
  - **Income Statistics** section restructured: now shows "Total Hotspot Generated Income" (from `used` + `running-active` vouchers) and "Expected Income" (from `waiting` vouchers) as separate cards
- **`includes/dash_connect.php`** — Replaced `/user-manager/payment/print` income source
  - Income now extracted by parsing price from profile names (e.g. `Tzs_1000_18H` → 1000 Tsh)
  - Added `/user-manager/session/print` call to derive today/week session counts and income
  - New `parseMikrotikDatetime()` helper handles both `jan/05/2026 08:30:00` and ISO datetime formats
  - API now returns `usersToday`, `usersThisWeek`, `incomeToday`, `incomeThisWeek`, `incomeTotalGenerated`, `incomeExpected`

### 2026-04-07 (update 2)

- **`accounting.php`** — New General Ledger page
  - **Overview tab**: P&L summary box (revenue, collections, commissions, net balance), monthly collections bar chart, quick agent commission status table with progress bars
  - **Ledger tab**: Chronological transaction table (sales + collections) with date range filter, running balance column, CSV export
  - **By Agent tab**: Agent P&L cards (vouchers sold, sales, commission due, collected, settlement progress bar) — updates live from RouterOS
  - **Collections Log tab**: Full collections table with agent filter search, add collection modal (agent, amount, datetime, note), delete entry with confirmation
  - Server-side PHP: reads `agentscollections.json`, `user_comments.json` (NDJSON), `agents.json`, `deleted_vouchers.json`; handles POST `add_collection` and `delete_collection`
  - JavaScript: loads live voucher data from `usage_api.php`, recomputes stats, updates all panels without page reload

### 2026-04-07

- **`voucher_export.php`** — New page: filter/export/print hub
  - Filter tabs: By Date, By Agent, By Status
  - Per-row and bulk checkbox selection
  - CSV export (client-side, UTF-8 BOM)
  - Excel export (client-side SpreadsheetML)
  - EPOS print modal (printer selector + copies)
  - **PDF Export modal** with Theme Manager + Settings Editor
- **`voucher_preview.php`** — New popup page
  - Scratch-pad rendering strategy: theme `renderVoucher()` called once per voucher, then cloned
  - Supports multi-voucher print-to-PDF and single-voucher Download PNG (html2canvas 3×)
  - Reads voucher data from `localStorage["vex_preview_vouchers"]`
- **`includes/voucher_export_api.php`** — New API
  - GET: voucher list with flattened meta fields
  - POST: `update_print`, `bulk_update_print`, `export_excel`
- **`includes/print_api.php`** — New API
  - ESC/POS byte generator (PHP, no extra lib)
  - Network printer via TCP socket; USB printer via `copy /b LPT1`
  - Updates RouterOS `print.*` comment fields after successful print
- **`includes/pdf_theme_api.php`** — New API
  - GET: theme discovery + defaults reader
  - POST: `set_default`, `save_defaults`
- **`includes/navbar.php`** — Added "Export & Print" link to Vouchers dropdown (desktop + mobile)
