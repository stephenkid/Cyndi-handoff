> snapshot of .kiro/reports/redemption-design.md as of 2026-11-26; 本文已按公开仓库规则做过脱敏（去掉密码、个人邮箱、利用细节）。
> the live copy lives in cyndi-global-website, check it for newer revisions.

---

# Custom Redemption Portal: Design (revision 4)

Status: design only, no code written. Client #1: Hudson River Trading (HRT; client admins Megan and Fei). Nothing here has been run, committed or pushed.
Migrations: `0014_client_scope.sql` and `0015_redemption.sql` (0013 is the latest applied).

What changed in this revision:
- The user's addendum (`.agents/tasks/redemption/design-addendum.md`) replaces the single-tenant, Cyndi-admin-only design. There are now three tiers (Cyndi admin, Showcase admin, storefront) and every Showcase data set is scoped by `client_id` (sections 2, 5, 8, 11, 15).
- The latest review (3 MEDIUM, 5 NIT) is fixed. Responses are in section 18.
- Addendum round 4: the three provisional inventory commits (c3c3943, 28a7a6f, 44f6370) are handled in the new section 8.7. The redemption feature does not depend on them.
- The ARCC query failed again (Midway 401, "run mwinit"). Section 10 is still provisional until it is re-run.

## 0. Overview

A "portal" belongs to one client. It holds a small pool of that client's Showcase products and a per-item quantity cap. A recipient opens a link with no login, picks items, types a shipping address and submits. The submission becomes a normal row in `orders` (prefix `RDM-`, `orders.client_id` set), with stock reserved in the Guangzhou inventory module at submission time.

| | Type A "personal" | Type B "open" |
|---|---|---|
| Link | One unique link per recipient (name + email list), emailed by the system | One shared link the client distributes |
| Approval | None. Order goes straight to `pending` | Every order is `awaiting_approval` until a Showcase admin of the client approves |
| Recipient email | Fixed from the list | Typed by the recipient |
| Reject | n/a (the order can be cancelled; the link works again) | Releases the reservation, order `cancelled` |
| Submissions | One live order per link | One live order per email per portal |

Both types share items, caps, wording, countries and fulfilment. No expiry (a portal can be paused or closed).

Who does what:
- Client admins (Fei, Megan: Showcase admins of HRT) create portals, import recipients, send emails, approve or reject, export CSV. They work in the Showcase app.
- Cyndi staff (existing `/admin`) fulfil `pending` redemption orders on the existing Orders and Shipments pages, manage stock, and have a read-only global Redemption page.
- Recipients use the public page `/r/<token>`.

Stack (locked once approved): Cloudflare Pages Functions (JS, ES modules), D1, vanilla HTML/JS for the public page and the Cyndi admin pages, React for the Showcase admin UI (the existing `showcase/` app, which already uses React/Vite/Tailwind), Resend HTTPS API for email, Cloudflare Turnstile for the open portal. No new npm dependency in either repo.

## 1. Requirements

### Functional
1. R1 A Showcase admin creates a portal of type `personal` or `open` for their own client, with name, public title, intro text, success text, item pool, per-item cap and optional country restriction.
2. R2 Per-item cap is a whole number (for example 1) or unlimited. Same for both types.
3. R3 Type A: the admin pastes or uploads name + email rows. The system creates one unique link per row and emails it. The admin can resend, void, copy the link and see per-recipient status (email sent, opened, redeemed).
4. R4 Type B: the admin gets one open link to copy. Every submission is held for review. The admin approves, or rejects with a note.
5. R5 The public page needs no login. The recipient enters name, phone, structured shipping address and country, and chooses colour/size/quantity of the offered items.
6. R6 No expiry. Ships to any allowed country (section 6.6).
7. R7 Stock: reserve at submission, reject when out of stock, release on reject or cancel, deduct on ship (existing flow).
8. R8 The admin can list claims and export recipients and claims as CSV. The order also appears in the Cyndi Orders and Shipments pages.
9. R9 The client can test the build and give feedback before go-live.
10. R10 Multi-client: every portal, order, product, category and catalog setting belongs to a client. A Showcase admin sees and changes only their own client's data.
11. R11 Showcase admin v1 modules: Orders, Inventory, Redemption. Nothing else.
12. R12 Cyndi admin can assign Showcase admins (a flag on a customer account of a client).

### Non-functional
- No other recipient's data and no stock numbers on the public page.
- Public endpoints validate every input, throttle and fail closed.
- Tenant isolation is enforced server side on every Showcase admin query. A guessed id from another client returns 404.
- Re-runnable SQL where possible. The three `ALTER TABLE ADD COLUMN` statements cannot be made re-runnable (section 5.3).
- Missing migration gives a clear 503 and no partial writes (section 6.5).

### Acceptance criteria
1. AC1 Anonymous requests (no cookie, garbage cookie, expired cookie) to every `/api/admin/redemption` GET and every `/api/showcase-admin/*` method return 401 and no data.
2. AC2 A valid Cyndi admin cookie sent to `/api/showcase-admin/*` returns 401. A valid customer cookie sent to any `/api/admin/*` returns 401.
3. AC3 A customer with Showcase access but no `client_admins` row gets 403 on `/api/showcase-admin/*`. Removing the row, disabling the customer, disabling the client, or moving the customer to another client takes effect on the next request.
4. AC4 With client A's admin session, every id belonging to client B (portal, code, claim, order number, product, SKU) returns 404 on every read and write op. A `clientId` in the query string or body is ignored.
5. AC5 Unknown public token returns 404 with the same body for every unknown token. After 10 failures from one IP in 10 minutes the response is 429.
6. AC6 A Type A link submits once. A second submit with a different idempotency key returns 409 `ALREADY_REDEEMED`. Ten concurrent submits on one link create exactly one order and reserve stock once.
7. AC7 A Type B submit creates an order with status `awaiting_approval` and reserves stock. It cannot be moved to `processing` or `shipped` from any orders API.
8. AC8 Rejecting a Type B order sets it `cancelled`, releases the reservation and writes a `release` movement. Approving sets `pending` with no stock change. Shipping deducts as today.
9. AC9 A submit above the cap, above stock, for an item outside the pool, or for a product of another client returns 400/409 and creates no order and no reservation.
10. AC10 A portal with an untracked variant cannot be activated, and a submit for an untracked variant is refused.
11. AC11 A repeated submit with the same idempotency key returns 200 with the first order number and creates no second order, for both types, including Type A after the first submit succeeded. If that first order was cancelled or rejected, the same key returns 409 `KEY_USED` and creates nothing.
12. AC12 Resend sends the same recipient's link again without creating a second link. A failed email shows `failed` with a short reason and can be retried.
13. AC13 Public API responses contain no `available/reserved` numbers, no other recipients' names or emails, no `stockSku`, no client id.
14. AC14 The Type A recipient email cannot be changed by the recipient.
15. AC15 CSV cells starting with `= + - @` (or tab/CR) are prefixed with `'`.
16. AC16 Every image URL in an anonymous `open` response returns 200 to an anonymous client.
17. AC17 `GET /r/<single path segment>` returns 200 with the page body (not a redirect) and the security headers. `/r/a/b` is not served by the function and returns the static 404.
18. AC18 Deleting an `RDM-` order is refused (409). The claim row and its address survive.
19. AC19 An invite stuck in `sending` for more than 10 minutes becomes eligible for resend, and two concurrent `sendEmails` calls never send the same row twice.
20. AC20 The Orders page and the Showcase admin orders API open, change status of, and cancel an `RDM-YYYYMMDD-XXXXXX` order (no "Invalid order number").
21. AC21 A Showcase admin can only cancel orders (from `pending` or `processing`). Any other status change, and any inventory write, returns 405 or 400.
22. AC22 After 0014, no `showcase_categories`, `showcase_products` or Showcase order (`SC-`/`SHOWCASE-`) row has `client_id IS NULL`. The storefront still shows HRT's catalog unchanged to HRT customers and nothing from other clients.
23. AC23 `node --test tests/` passes, `npm run lint` in `product-showcase` and in `showcase` passes, `npx vite build` passes in both.
24. AC24 Open portals always require captcha. `createPortal` and `updatePortal` ignore any `captchaRequired` input, and the database refuses an open portal with `captcha_required = 0`.
25. AC25 `setStatus('active')` on an open portal returns 409 `Captcha is not configured` when `TURNSTILE_SITE_KEY` or `TURNSTILE_SECRET_KEY` is missing, except when `REDEEM_MAIL_MODE=log` (local demo, where the page uses Cloudflare's test keys from `.dev.vars`). A submit to an open portal without the keys returns 503.
26. AC26 `GET /api/redeem?media=<id>` returns 200 only for an id that exactly equals the image URL of a pool product (or one of its colours) of an active portal of an active client. A longer id that merely contains it, a gallery-only image, and an image of a draft/paused/closed portal return 404.
27. AC27 Reset of stock for one client leaves every other client's inventory, movements and reservations unchanged, and is refused while that client has an open reservation from another order (section 8.7).

### Out of scope (v1)
Client branding; client-creation wizard (SQL path only); recipient emails after submit, approval, reject or ship; approver notification emails; portal expiry; auto-release of old pending reservations; per-variant restrictions inside an item; ZIP/label printing; payment; recipient tracking page; Showcase admin writes to inventory; Showcase admin catalog editing; per-role permissions inside a client; sandbox portals; email delivery webhooks; PII purge job; Cyndi-admin approval on behalf of a client.

### Assumptions
- A1 Fei and Megan each get their own customer account (own email, client HRT, `showcase_access = 1`) created on the existing Customers page, then are flagged Showcase admin. They are not in `admin_users` (the `scripts/make_admin_user.mjs` path is unused for this).
- A2 Redemption items are Showcase products with inventory in `inventory`.
- A3 Redemption orders carry no price (`total = 0`).
- A4 `CUSTOMER_SESSION_SECRET` is set in production (PROGRESS.md says it is).
- A5 `env.ASSETS.fetch('/redeem')` works from a Pages Function for the `/r/*` route (verified in the test plan).
- A6 All existing Showcase customers belong to a client today (checked by a pre-flight query, section 15.1).

## 2. Architecture: three tiers

| Tier | Identity | Where it lives | API namespace | Sees |
|---|---|---|---|---|
| Cyndi admin | `admin_users`, cookie `logistics_admin` (aud `admin`) | `product-showcase/public/admin/*.html`, `functions/api/admin/*` | `/api/admin/*` | Everything |
| Showcase admin | Customer account (`customers`) with a `client_admins` row, cookie `customer_session` (aud `customer`) | `showcase/` repo (React) | `/api/showcase-admin/*` (new) | Own client only: Orders, Inventory (read-only), Redemption |
| Storefront / recipients | Customer session / none | existing Showcase, public `/r/<token>` | `/api/showcase/*`, `/api/redeem` | Own orders / own link |

Separation:
- `requireAdmin` accepts only `aud: 'admin'` tokens (`readSession`, auth.js:118). The customer session only accepts `aud: 'customer'`. The two cookies cannot be exchanged (AC2).
- A Showcase admin is never in `admin_users` and no `/api/admin/*` route accepts a customer cookie.

### 2.1 `requireShowcaseAdmin(request, env)` in `functions/_lib/showcase-admin-auth.js`

Order of checks, each failing closed:
1. `env.DB` present, else 503.
2. `readCustomerSession` (customer-auth.js:309): not configured is 503, no session is 401. It loads the customer fresh from D1 on every call, so a disabled customer is rejected at once.
3. `hasShowcaseAccess(customer)` (active, `showcase_access = 1`, client active), else 403 `This account does not have Showcase access`.
4. `customer.client_id` is not NULL, else 403.
5. `SELECT 1 FROM client_admins WHERE customer_id = ?1 AND client_id = ?2`, with both values from the loaded customer row. No row: 403 `Not a Showcase admin`.
6. For `POST`: `Content-Type: application/json`, and if an `Origin` header exists it must equal the request origin, and if `Sec-Fetch-Site` exists it must be `same-origin`. Reason: the customer cookie is `SameSite=Lax`, not `Strict` (customer-auth.js:321). A JSON content type already forces a CORS preflight and no CORS headers exist. The Origin check is extra protection.
Returns `scope = { clientId, customerId, actor, label }` where `actor = 'showcase-admin:<customerId>'` and `label` is the display name. `clientId` comes only from the DB row. Query and body fields named `clientId` are never read.

Every response from this namespace: `Cache-Control: private, no-store`, `Vary: Cookie`, no CORS headers.

### 2.2 Scope rule for data access

Every statement that reads or writes a client-owned table in the Showcase admin namespace includes the client id as a bound parameter. The invariant is owned by the shared libraries (section 8.2) so a route file cannot forget it:
- Lookups by id use helpers that take `scope`: `rdmLoadPortal(db, id, scope)`, `rdmLoadClaim(db, id, scope)`, `ordLoadDetail(db, no, scope)`, `invLoadProductScoped(db, productId, scope)`. They return `null` for another client's row, and the route answers 404 (not 403, so existence is not revealed).
- Lists add `AND <alias>.client_id = ?`.
- A static test (section 14.1) scans the scoped modules and fails if a SQL statement on a client-owned table has no `client_id` and is not on an explicit allowlist.

`scope.clientId === null` is allowed only in read paths used by Cyndi admin. Every write function throws if `scope.clientId` is not a positive integer.

## 3. Existing code that shapes the design

- `orders` has no address columns and `status` has no CHECK. Order source is told apart by `order_no` prefix (`SW-`, `SHOWCASE-`, `SC-`). `orders.customer_id` links a customer, and the client is only reachable through `customers.client_id`. Redemption orders have no customer, so `orders.client_id` is needed (5.2).
- `showcase_products.id` and `showcase_categories.id` are global text primary keys. `showcase_meta` is a single row (`id = 1`). `inventory.sku` is the primary key, `UNIQUE (product_id, color_id, size)`, no FK to products (0013).
- `inventory.product_id` is a product id, and a product id belongs to exactly one client (section 5.4), so inventory is already per client through the product.
- Public catalog `functions/api/showcase.js` and `admin/showcase.js` select all products and all inventory with no filter. `showcase/orders.js:292` loads products by id with no client filter. Runtime `ensureSchema` DDL and `seedIfEmpty` exist in `api/showcase.js` and `admin/showcase.js`, and the seed does `INSERT OR REPLACE` without a client.
- `admin/orders.js` has its own `isValidOrderNo` (lines 22-23, used at 136 and 342). `admin/shipments.js:24` and `admin/shipments/upload.js:16` have copies. All three reject `RDM-`.
- `admin/orders.js` applies stock effects only when `isShowcaseOrderNo(no)`. `prepareInsertOrder` hardcodes status `pending`. `order-status.js` has no `awaiting_approval`.
- `functions/api/orders.js` (public lookup) and `orders/mine.js` allow only `SW-`/Showcase prefixes, so `RDM-` is already a 404 there.
- `admin/inventory.js` is a set of functions that take `db` (`listProducts`, `productDetail`, `loadOpenOrders`, `opTrack`, `opSet`, `opAdjust`, `opThreshold`) plus two route handlers.
- No email code, no rate-limit code, no `_headers`, `_redirects` or `_routes.json` exists. Only the root `functions/` directory is live.
- Pages bundles `_lib` modules into one namespace, so new exports use prefixes `rdm`, `mail`, `sadm` (showcase admin) and `ord`.
- Customers page (`admin/customers.js`) already creates customers, assigns `client_id` and `showcase_access`, and lists clients.

## 4. Decisions on the open points (all repeated in section 16)

| # | Open point | Recommendation |
|---|---|---|
| D1 | How redemption orders are stored | Reuse `orders` (prefix `RDM-`, `orders.client_id` set) plus a 1:1 `redemption_claims` table with the structured address and approval state. No `order_type` column. |
| D2 | Structured address | Only in `redemption_claims`. A general `order_addresses` table for other orders is a separate task. |
| D3 | Approval state | New order status `awaiting_approval` plus `redemption_claims.approval_status`. |
| D4 | Code/link format | 128-bit random token, 22 chars base64url, link `https://<site>/r/<token>`. Personal tokens stored in plain text (needed for resend). One shared token per open portal. |
| D5 | Anti-abuse | D1 throttle per IP hash, Turnstile on open-portal submit, unique email per open portal, soft `max_total_orders`, pause and rotate buttons, optional WAF rule by hand. |
| D6 | Item pool and cap | `redemption_portal_items(portal_id, product_id, max_qty NULL = unlimited)`. Cap is per product per order, summed over colours and sizes. |
| D7 | Single vs multiple submissions | Type A: one live order per link. Type B: one live order per email. Both enforced by unique indexes. A cancelled or rejected order frees the slot. |
| D8 | Email provider | Resend. Invites only in v1. Daily per-client send cap. |
| D9 | Wording storage | Plain-text columns on `redemption_portals`, length capped, rendered with `textContent`. |
| D10 | Roles | One role: Showcase admin of a client (all of the three modules). Cyndi `admin_users` roles stay separate and untouched (section 11). |
| D11 | Public page location | Static page in the main site (`product-showcase/public/redeem.*`). The Showcase admin UI is in the `showcase/` repo. |
| D12 | Tenant model | Add `client_id` to products, categories and orders. Inventory, stock lines and movements are scoped through the product / order, with no new column (5.4). Catalog settings move to `showcase_meta_client`. |
| D13 | Showcase admin write rights | Orders: view and cancel. Inventory: read-only. Redemption: full. |
| D14 | Cyndi admin and redemption | Read-only global page. No approve/reject by Cyndi staff in v1. |

## 5. Data model

### 5.1 Migration files and how the user applies them

| File | Content | Re-runnable |
|---|---|---|
| `migrations/0014_client_scope.sql` | Section A: 3 `ALTER TABLE ... ADD COLUMN client_id`. Section B: new tables `client_admins`, `showcase_meta_client`, indexes, and backfill `UPDATE`s / `INSERT OR IGNORE`. | Section B yes. Section A no: a second run fails on the first ALTER with "duplicate column name", which means it is already applied. Paste Section A statement by statement and skip any that report that error. |
| `migrations/0015_redemption.sql` | Five redemption tables and indexes, all `CREATE ... IF NOT EXISTS`. | Yes |

Header comment in both files uses the 0013 style (Chinese + English one-liner, "do not use `migrations apply --remote`"). Local: `npx wrangler d1 execute cyndi-logistics --local --file=migrations/0014_client_scope.sql`, then the same for 0015. A second local run of 0014 stops at the first ALTER; use a fresh local DB or paste Section B only. Never `wrangler d1 ... --remote`. The user pastes each file into D1 Studio Console (section 15 gives the order).

### 5.2 Columns added by 0014 Section A

```
ALTER TABLE showcase_categories ADD COLUMN client_id INTEGER;
ALTER TABLE showcase_products   ADD COLUMN client_id INTEGER;
ALTER TABLE orders              ADD COLUMN client_id INTEGER;
```
All three are nullable at the database level because SQLite cannot add a NOT NULL column without a constant default, and the HRT client id is not a constant. App code requires a value on every insert. A later table rebuild can add NOT NULL once the app has run for a while (listed under risks).

`orders.client_id` meaning: the client whose storefront or portal produced the order. `SW-` shop orders stay NULL (they are not client orders and only Cyndi admin sees them).

### 5.3 Tables and backfill in 0014 Section B

```
client_admins(
  customer_id INTEGER PRIMARY KEY REFERENCES customers(id) ON DELETE CASCADE,
  client_id   INTEGER NOT NULL REFERENCES clients(id),
  granted_by  TEXT NOT NULL,          -- 'admin:<username>'
  created_at  TEXT NOT NULL)
CREATE INDEX IF NOT EXISTS idx_client_admins_client ON client_admins(client_id);

showcase_meta_client(                 -- replaces the single-row showcase_meta
  client_id  INTEGER PRIMARY KEY REFERENCES clients(id),
  data_json  TEXT NOT NULL,
  updated_at TEXT NOT NULL)
```
Backfill, each guarded so a re-run changes nothing:
1. `UPDATE showcase_categories SET client_id = (SELECT id FROM clients WHERE code='HRT') WHERE client_id IS NULL;`
2. Same for `showcase_products`.
3. Orders of customers: `UPDATE orders SET client_id = (SELECT c.client_id FROM customers c WHERE c.id = orders.customer_id) WHERE client_id IS NULL AND customer_id IS NOT NULL AND (order_no LIKE 'SC-%' OR order_no LIKE 'SHOWCASE-%');`
4. Remaining Showcase orders with no customer: `UPDATE orders SET client_id = (SELECT id FROM clients WHERE code='HRT') WHERE client_id IS NULL AND (order_no LIKE 'SC-%' OR order_no LIKE 'SHOWCASE-%');`
5. `INSERT OR IGNORE INTO showcase_meta_client (client_id, data_json, updated_at) SELECT (SELECT id FROM clients WHERE code='HRT'), data_json, updated_at FROM showcase_meta WHERE id = 1;`
6. Indexes: `idx_showcase_products_client(client_id, category_id, sort)`, `idx_showcase_categories_client(client_id, sort)`, `idx_orders_client(client_id, created_at)`.

`showcase_meta` stays in place unused after the code switch (kept for rollback).

Why `client_admins` is a separate table and not a column on `customers`: the grant is a Cyndi-admin decision with its own audit fields, a flag on `customers` would be edited by the generic customer update op, and the table can later get a `role` column without touching `customers`.

### 5.4 Why inventory gets no `client_id`

The addendum asks for inventory per client, "client + product + colour + size, or equivalent". The equivalent already exists:
- `inventory.product_id` references a product, and `showcase_products.id` is a global primary key. A product therefore has exactly one client, and `UNIQUE (product_id, color_id, size)` is already unique per client.
- `inventory.sku` is the primary key and `invSkuCode` de-duplicates against all existing codes with `LIKE` (admin/inventory.js:387), so two clients that stock the same supplier style get `...-2` on the second. Codes never collide. The cost is a cosmetic suffix.
- A second `client_id` column on `inventory`, `order_stock_lines` and `inventory_movements` would copy a value that must always equal `showcase_products.client_id`. Nothing in SQLite enforces that without triggers, and drift would silently break tenant isolation. Scoping is done by a join to the product (`inventory`) or to the order (`order_stock_lines`, via `orders.client_id`). Movements are scoped by `sku IN (SELECT sku FROM inventory JOIN showcase_products ... WHERE client_id = ?)`.
- Rule that keeps this safe: a product's `client_id` never changes. `saveProduct` refuses to change it (section 8.5). No op moves a product to another client.
- Orphans: inventory rows whose product was deleted have no client, so they are visible to Cyndi admin only (the Inventory page already lists them, inventory.js:166). Accepted.
- Alternative if the user prefers: add `client_id` to `inventory` only, with a backfill and a check in `opTrack`. It costs one more ALTER and a write-path invariant. Flagged in section 16, decision 12.

### 5.5 `migrations/0015_redemption.sql`

All five tables carry `client_id`. Child tables use a composite FK `(portal_id, client_id) → redemption_portals(id, client_id)`, so the database refuses a child row whose client differs from its portal's. `redemption_portals` has `UNIQUE (id, client_id)` for that purpose.

#### `redemption_portals`

| Column | Type / rule | Notes |
|---|---|---|
| id | TEXT PK | `rp_` + 10 random `[a-z0-9]`. Not secret. |
| client_id | INTEGER NOT NULL, FK clients(id) | Owner. UNIQUE (id, client_id) |
| type | TEXT NOT NULL, CHECK IN ('personal','open') | |
| name | TEXT NOT NULL | Internal, 1..100 |
| status | TEXT NOT NULL DEFAULT 'draft', CHECK IN ('draft','active','paused','closed') | Only `active` is reachable publicly |
| title | TEXT NOT NULL | Public heading, 1..120 |
| intro_text | TEXT | Public, ≤2000, plain text with line breaks |
| success_text | TEXT | Public, ≤1000 |
| email_subject | TEXT | Type A, ≤150 |
| email_intro | TEXT | Type A, ≤2000, placed above the button. Shown as plain text, URLs not made clickable |
| requires_approval | INTEGER NOT NULL | CHECK `(type='open') = (requires_approval=1)`. The database enforces "Type B always needs approval" |
| open_token | TEXT UNIQUE | Type B only. CHECK `(type='open') = (open_token IS NOT NULL)` |
| countries_json | TEXT | NULL = default list; else JSON array of ISO-3166 alpha-2, subset of the default list |
| captcha_required | INTEGER NOT NULL DEFAULT 0, CHECK `(type <> 'open' OR captcha_required = 1)` | Open portals always require captcha: the API writes 1 for `open` and the CHECK rejects 0. Not an API input and no UI toggle. Personal portals write 0. The column is kept so a personal portal could opt in later |
| max_total_orders | INTEGER, CHECK NULL or ≥1 | Soft cap on live orders (see below). UI default 200 for open portals |
| created_by | TEXT | `showcase-admin:<customerId>` |
| created_at, updated_at | TEXT | ISO timestamps |

`max_total_orders` is a soft cap. The submit path reads the live count and then inserts, so concurrent submits can overshoot by the number of requests in flight. The overshoot is bounded by the `portal-submit` throttle (60 per hour) and the admin reviews every Type B order anyway. The admin UI says "about N orders". A hard cap would need the claim insert to be conditional, but the order insert and stock reservation in the same batch cannot be rolled back by a conditional insert that affects zero rows, so the soft cap is the chosen design.

`client_label` from earlier drafts is removed: the portal's client name is `clients.name`, copied into `orders.company`.

#### `redemption_portal_items`

PK (portal_id, product_id). `client_id` INTEGER NOT NULL. FK `(portal_id, client_id)` → portals, ON DELETE CASCADE. `product_id` = `showcase_products.id` (no FK, same reason as `inventory`; the app checks that the product's `client_id` equals the portal's on create, update, activate and submit). `max_qty` INTEGER NULL (NULL = unlimited), CHECK NULL or 1..100. `sort` INTEGER NOT NULL DEFAULT 0. "Unlimited" still has a hard limit of 100 per product per order.

#### `redemption_codes` (Type A recipients)

| Column | Type / rule |
|---|---|
| id | INTEGER PK AUTOINCREMENT |
| portal_id, client_id | TEXT / INTEGER NOT NULL, composite FK to portals ON DELETE CASCADE |
| token | TEXT NOT NULL UNIQUE |
| name | TEXT NOT NULL (≤100) |
| email, email_norm | TEXT NOT NULL (`email_norm` trimmed lower-case). UNIQUE (portal_id, email_norm) |
| status | TEXT NOT NULL DEFAULT 'active', CHECK IN ('active','void') |
| email_status | TEXT NOT NULL DEFAULT 'not_sent', CHECK IN ('not_sent','sending','sent','failed') |
| send_count | INTEGER NOT NULL DEFAULT 0 |
| email_sent_at, first_opened_at, sending_at | TEXT. `sending_at` set when a row is claimed; `sending` older than 10 minutes counts as `failed` |
| email_error | TEXT (≤200, provider message, no secrets) |
| created_at | TEXT NOT NULL |

"Redeemed" is derived from a live claim, not stored here.

#### `redemption_claims` (one row per submission, 1:1 with an order)

| Column | Type / rule |
|---|---|
| id | INTEGER PK AUTOINCREMENT |
| order_no | TEXT NOT NULL UNIQUE, FK orders(order_no) ON UPDATE CASCADE ON DELETE RESTRICT |
| portal_id, client_id | composite FK to portals (no cascade; a portal with claims cannot be deleted, only closed) |
| code_id | INTEGER, FK codes(id) (NULL for open portals) |
| idem_key | TEXT NOT NULL (16..64 chars) |
| recipient_name, recipient_email, email_norm, phone | TEXT NOT NULL |
| addr_line1 NOT NULL, addr_line2, city NOT NULL, region, postal_code | TEXT |
| country | TEXT NOT NULL (ISO alpha-2) |
| delivery_note | TEXT (≤300) |
| approval_status | TEXT NOT NULL, CHECK IN ('not_required','pending','approved','rejected') |
| reviewed_by_id INTEGER, reviewed_by TEXT, reviewed_at, review_note | `reviewed_by` = reviewer's email at the time, `review_note` ≤500, internal only |
| counts | INTEGER NOT NULL DEFAULT 1. 1 while the order is live, 0 after cancel/reject |
| ip_hash | TEXT (HMAC of the client IP, not the raw IP) |
| submitted_at | TEXT NOT NULL |

Indexes:
- UNIQUE (portal_id, idem_key): double click / retry safety (AC11). The submit path looks the key up before the live-claim check (6.4 step 3).
- UNIQUE (code_id) WHERE `counts = 1 AND code_id IS NOT NULL`: one live order per personal link (AC6).
- UNIQUE (portal_id, email_norm) WHERE `counts = 1 AND code_id IS NULL`: one live order per email on an open portal.
- (client_id, approval_status, submitted_at): lists and the pending badge.

`ON DELETE RESTRICT` keeps the claim, its address and `idem_key` from vanishing with an order. The delete ops also refuse `RDM-` orders up front (8.5).

`counts` exists because a unique index cannot look at `orders.status`. It is flipped to 0 in the same batch that cancels or rejects the order.

#### `redemption_throttle`

PK (key, window_start). `key` TEXT (`<action>:<hash or id>`), `window_start` INTEGER (unix seconds), `count` INTEGER NOT NULL. No tenant data. Each throttle write has a 1% chance to run `DELETE ... WHERE window_start < now - 86400`.

### 5.6 Rows of existing tables written by a claim

- `orders`: `order_no` `RDM-YYYYMMDD-XXXXXX` (6 random chars from `crypto.getRandomValues`, regex `^RDM-\d{8}-[A-Z0-9]{6}$`), `customer_name`, `customer_email`, `customer_phone` from the recipient, `company` = `clients.name`, `notes` = `[redeem:<portal id>] <delivery note>`, `total` = 0, `customer_id` NULL, `client_id` = portal's client, `status` = `awaiting_approval` or `pending`.
- `items_json` lines: `productId, sku, name, colorIndex, colorName, colorHex, size, quantity, image, stockSku, unitPrice: 0, lineTotal: 0, currency: null`. `currency: null` makes `itemsCurrency()` return `null` (admin/orders.js:52). The Orders page shows redemption totals as "No charge" (small tweak in `orders.html`).
- `order_stock_lines`, `inventory`, `inventory_movements`: written by `invReserveStatements` exactly as for Showcase orders.

## 6. Public flow (recipient)

### 6.1 Files

| File | Purpose |
|---|---|
| `functions/r/[token].js` | `GET /r/<token>` (a single path segment). Fetches `/redeem` through `env.ASSETS.fetch` (the clean URL, because Pages answers `/redeem.html` with a 308), builds a new `Response` from the body with fresh headers, and returns 500 generic if the asset fetch is not 200. Sets `Cache-Control: no-store`, `Referrer-Policy: no-referrer`, `X-Robots-Tag: noindex, nofollow` and a CSP. No DB access, so a bad token costs nothing. `/r/a/b` is not matched and falls through to the static 404. |
| `product-showcase/public/redeem.html`, `redeem.js`, `redeem.css` | Vanilla page. All text via `textContent`. External script/CSS so the CSP needs no `unsafe-inline`. |
| `functions/api/redeem.js` | The only public data endpoint. `POST` for `open` and `submit`. `GET` only as `GET /api/redeem?media=<id>`. Other `GET` and `OPTIONS` return 405. No CORS headers. |
| `functions/_lib/redemption.js` | Pure helpers (`rdm*`): tokens, validators, countries, snapshot builder, statement builders, error classifier, CSV escape, import parser, throttle. |
| `functions/_lib/mailer.js` | `mail*`: Resend client, message builders. |

CSP for `/r/*`: `default-src 'none'; script-src 'self' https://challenges.cloudflare.com; style-src 'self'; img-src 'self' <R2 origin> data:; connect-src 'self'; frame-src https://challenges.cloudflare.com; base-uri 'none'; form-action 'none'; frame-ancestors 'none'`. `<R2 origin>` is the origin of `env.R2_PUBLIC_BASE` (via `new URL`), added only when set. The Turnstile script is loaded by `redeem.js` only when `open` returns a `captcha` object.

The token travels in the POST body, never in an API query string, and `Referrer-Policy: no-referrer` stops it leaking to the image host. Fallback if `env.ASSETS` routing misbehaves: `public/_redirects` with `/r/* /redeem 200` plus a `public/_headers` file.

### 6.2 API: `POST /api/redeem` (JSON, ≤ 8 KB, `Content-Type: application/json` required)

All responses `Cache-Control: no-store`. Errors are `{error, code?}`, generic, with no stock numbers.

**op `open`** `{op, token}`
1. Throttle `open-fail` peek.
2. Look up `redemption_codes.token`, else `redemption_portals.open_token`. Not found: count a failure, 404 `{error:'This link is not valid'}`.
3. Portal not `active`, its client not `active`, or code `void`: 410 `{error:'This link is not available right now', code:'UNAVAILABLE'}`.
4. Type A with a live claim: 200 `{state:'redeemed', orderNo, title}`.
5. Otherwise 200:
```
{ state:'open', kind:'personal'|'open', requiresApproval,
  recipient: {name}            // personal only; email never returned
  portal: {title, introText, successText},
  items: [{productId, name, image, maxQty|null,
           colors:[{index,name,hex,image}], sizes:[...],
           variants:[{colorIndex|null, size|'', inStock:true|false}]}],
  countries: [{code,name}],
  captcha: {siteKey} | null }   // open portals: always an object. personal: null
```
For an open portal with `TURNSTILE_SITE_KEY` unset, `open` returns 503 `{error:'Redemption is not available'}`, so the page never shows a form that cannot be submitted.
`inStock` is `available > 0` only. Variants without an inventory row are not listed. An item with no orderable variant is dropped. If none remain, `open` returns 410 `UNAVAILABLE` and the admin list shows a warning. Products are loaded with `showcase_products.client_id = portal.client_id`, so a product that was somehow added from another client is dropped. Images pass through `rdmPublicImage` (6.2a). `first_opened_at` is set once for Type A.

**op `submit`**
```
{ op, token, idempotencyKey,
  contact: {name, phone, email?},          // email required for open, ignored for personal
  address: {line1, line2?, city, region?, postalCode?, country},
  items: [{productId, colorIndex?, size?, quantity}],
  deliveryNote?, captchaToken?, website }  // website = honeypot, must be ""
```

### 6.2a Images

Product images are static paths (`/showcase/images/...`), R2 public URLs under `R2_PUBLIC_BASE`, or `/api/showcase/media?id=<id>` (login-gated by `requireShowcaseCustomer`, so an anonymous recipient would get 401). `rdmPublicImage(url)` runs on every image in the `open` response:
- Static path, or URL whose origin equals the `R2_PUBLIC_BASE` origin: unchanged.
- `/api/showcase/media?id=<id>` (id matches `^[A-Za-z0-9_-]{1,64}$`): rewritten to `/api/redeem?media=<id>`.
- Anything else (other host, `data:`, `javascript:`): `null`, and the page shows a text placeholder.

`GET /api/redeem?media=<id>` serves one image to an anonymous caller. Steps:
1. Validate `<id>` with `^[A-Za-z0-9_-]{1,64}$`, else 404. Count the `media` throttle first.
2. In JS build `url = '/api/showcase/media?id=' + id` (the id has no characters that need encoding, so it equals what `encodeURIComponent` stored in `upload.js:46`).
3. Run this single parameterized statement with binds `?1 = id`, `?2 = url`:
```
SELECT m.content_type, m.image_data
FROM showcase_media m
WHERE m.id = ?1 AND EXISTS (
  SELECT 1
  FROM redemption_portals rp
  JOIN clients cl                  ON cl.id = rp.client_id AND cl.status = 'active'
  JOIN redemption_portal_items ri  ON ri.portal_id = rp.id AND ri.client_id = rp.client_id
  JOIN showcase_products p         ON p.id = ri.product_id AND p.client_id = rp.client_id
  WHERE rp.status = 'active'
    AND ( p.image = ?2
          OR EXISTS (SELECT 1
                     FROM json_each(CASE WHEN json_valid(p.colors_json) THEN p.colors_json ELSE '[]' END) j
                     WHERE json_extract(j.value, '$.image') = ?2) ) )
```
Both comparisons are whole-string equality. There is no `INSTR` or `LIKE`, so an id that is a prefix, suffix or substring of another stored URL matches nothing. `json_each` over `colors_json` reads the `image` key of each colour object whatever the key order is. Stored shape of a colour (checked on a local row; the `/showcase/images/...` value shown there is a static path, the media form is what `admin/showcase/upload.js` writes): `{"name":"Black","hex":"#1c1917","image":"<url>","id":"c_jljjs1"}`. `gallery_json` and `size_chart` are not read, so gallery-only images are not public.
4. No row: 404. Row found: reply with `Cache-Control: public, max-age=3600`, `X-Content-Type-Options: nosniff`, and a content type that must be one of `image/jpeg|png|webp|gif` (anything else: 404).

The statement is run only with ids that pass the regex. `p.image` of a draft, paused or closed portal's product is not reachable. Gallery images and size charts are not offered on the public page.

### 6.3 Validation (server side; client checks are UX only)

| Field | Rule | On failure |
|---|---|---|
| Body | JSON object, ≤ 8 KB, content type JSON | 415 / 413 / 400 |
| token | string, 22 chars `[A-Za-z0-9_-]` | 404 (same as unknown) |
| idempotencyKey | required, 16..64 chars `[A-Za-z0-9_-]` | 400 |
| website | empty string or absent | 400 generic, counted as a failure |
| contact.name | required, 1..100, no control characters | 400 with field message |
| contact.phone | required, 5..30 chars of `0-9 + ( ) - .` and space | 400 |
| contact.email | open: required, ≤254, matches `EMAIL_RE` from `functions/_lib/customer-auth.js`, no control characters. personal: ignored, taken from the code row | 400 |
| address.line1 / city | required, 1..120 / 1..80 | 400 |
| address.line2 / region / postalCode | optional, ≤120 / ≤80 / ≤20 | 400 |
| address.country | required, in the portal's allowed list | 400 |
| deliveryNote | optional, ≤300 | 400 |
| items | 1..20 lines. `colorIndex` and `size` normalised to `null` when absent or empty (`invResolveOrderLines` tests `=== null`), lines numbered `n` from 1. `productId` in the portal pool. `quantity` integer ≥ 1. `colorIndex`/`size` must exist on the product. Lines of the same product are summed and compared with `max_qty` (or 100) before the call | 400 |
| captchaToken | required when `captcha_required = 1`. Verified with Turnstile `siteverify` (secret from env, remote IP passed) | 400 `CAPTCHA`; 503 if the secret is missing |

All text is trimmed. Control characters (except newline in notes) are rejected, not stripped.

### 6.4 Submit sequence

1. Method/size/content-type checks. `DB` and `CUSTOMER_SESSION_SECRET` present, else 503 `{error:'Redemption is not available'}`. Syntax check of `token` and `idempotencyKey` (6.3); a bad token counts an `open-fail` and returns 404.
2. Resolve the token (unknown: count `open-fail`, 404). Portal `active`, client `active`, code `active`, else 410 `UNAVAILABLE`.
3. **Idempotent replay lookup**: `SELECT order_no, counts, approval_status FROM redemption_claims WHERE portal_id = ?1 AND idem_key = ?2`.
   - Found with `counts = 1`: return 200 `{ok:true, orderNo, requiresApproval, successText}` for that claim. No throttle count, no writes, nothing below runs. This covers the double click and the network retry for both types.
   - Found with `counts = 0` (the order was cancelled or rejected): 409 `{code:'KEY_USED'}`. The page generates a new key and the user can submit again (see "Idempotency key lifetime" in 6.5).
   - Not found: continue. The lookup is one indexed read and needs a valid token, so it adds no new abuse route.
4. Live-claim check (Type A): a live claim for this code returns 409 `ALREADY_REDEEMED`. This runs after step 3, so a retry with the same key never reaches it, and a second submit with a different key is refused (AC6).
5. Per-IP `submit` throttle, before any heavy work and before body validation. Over the limit: 429 with `Retry-After`.
6. Captcha (open portals, always). Missing `TURNSTILE_*` keys: 503. Fail: 400 `CAPTCHA`.
7. Per-portal `portal-submit` throttle (open portals), counted only after the captcha passed. Then the `max_total_orders` pre-check (soft cap, 5.5).
8. Validate the body (6.3).
9. Load the pool products (`showcase_products` joined to `showcase_categories`, both with `client_id = portal.client_id`) and `invLoadForProducts`. Run `invResolveOrderLines`. A line with `stockSku = null` is refused with 409 `UNAVAILABLE` and logged at `error` (admin mis-config). Shortages return 409 `OUT_OF_STOCK` with a number-free message. `invStockShortMessage` is not used publicly because it prints counts.
10. Build the order snapshot (no prices) and the claim fields.
11. One `db.batch`: `prepareInsertOrder(... status, clientId)`, `invReserveStatements` (3 statements), the `redemption_claims` INSERT. Order number collisions retry up to `INV_ORDER_NO_ATTEMPTS`.
12. Success: 200 `{ok:true, orderNo, requiresApproval, successText}`.

### 6.5 Error mapping for the batch

| Cause | Detected by | Result | Recoverable | Log |
|---|---|---|---|---|
| Stock raced to zero | `invClassifyError` = `insufficient` | 409 `OUT_OF_STOCK`, batch rolled back | yes | warn |
| Stock row vanished | `fk` | 409 `UNAVAILABLE` | yes | warn |
| Order number taken | `order_no_taken` | retry ≤5, then 500 | yes | warn / error |
| Duplicate `idem_key` (two requests with one key racing past step 3) | UNIQUE on `idem_key` | re-run the step 3 lookup: `counts = 1` returns 200 with that order number, `counts = 0` returns 409 `KEY_USED`. The batch is rolled back, so stock was reserved once | n/a | info |
| Link already used (race past step 4) | UNIQUE on `code_id` | 409 `ALREADY_REDEEMED` | no | info |
| Email already used (open) | UNIQUE on `email_norm` | 409 `EMAIL_USED` "We already have a request for this email" | no, until rejected | info |
| Table or column missing | `rdmClassifyError` = `missing_schema`: `no such table` for `inventory`, `order_stock_lines`, `inventory_movements`, `redemption_*`, or `no such column: client_id`. Also checked at step 7: if the pool has products and `invLoadForProducts` returns `[]`, probe `SELECT 1 FROM inventory LIMIT 1` | public 503 generic, admin 503 naming the migration | fatal until migrated | error |
| Anything else | catch | 500 `{error:'Could not submit. Please try again.'}` | yes | error, message only, no PII |

The three UNIQUE errors are told apart by the index/column text in the D1 error message. The exact text of partial-index violations is a build-time check: provoke each once on local D1 and use the recorded strings.

Edge cases: portal paused between page load and submit (410); product deleted after portal creation (item dropped from `open`; a submit with it is 400); colour or size removed (400); a cap lowered while the form is open (400 with the current cap); reopening a link after cancel (works); double click or reload (see below); client disabled (410).

**Idempotency key lifetime** (`redeem.js`): the page generates the key (16 random bytes, base64url) on load and keeps it in `sessionStorage` under `rdm-idem:<token>` so a reload keeps it. The page deletes the stored key after any 200, and after a 409 `ALREADY_REDEEMED` or `KEY_USED`, and generates a fresh one before the next submit. A network error or a 5xx keeps the key, so the retry is safe. Result: after a cancel (Type A) or a reject (Type B) the recipient can reopen the link in the same tab and submit again without getting the old order number back.

### 6.6 Countries

A constant list in `_lib/redemption.js`: ISO-3166 alpha-2 codes with English names, minus a deny list (CU, IR, KP, SY, RU, BY; regions such as Crimea cannot be expressed by country, so address review catches them). A portal can narrow it with `countries_json`. The HRT v1 doc lists SG, IN, HK, CN, AU, while the client now says "any country you can ship to". Flagged for confirmation (section 16).

## 7. Email

### 7.1 What exists and what to use

Nothing sends email today (no smtp, resend, mailchannels or sendgrid anywhere under `functions/`). Pages Functions cannot open SMTP sockets, so the provider needs an HTTPS API.

| Option | Verdict |
|---|---|
| **Resend** (HTTPS JSON `POST /emails`, `Idempotency-Key`, domain verification with SPF/DKIM) | Recommended. Smallest code, one secret. |
| Amazon SES | Needs SigV4 signing code and an AWS account. More code, no benefit at this volume. |
| Aliyun DirectMail | The company already uses Aliyun mail, but the HTTP API needs custom signature code. Fallback. |
| MailChannels | No longer a free route for Pages Functions. Not recommended. |

The Resend details above are general knowledge, not read from this repo. **Build step 5a (before writing `mailer.js`)**: read Resend's current docs and confirm the `POST /emails` request and response, the `Idempotency-Key` header and retention, domain verification steps and Workers `fetch` usage. If anything differs, update this section first. The design uses single sends only (no batch endpoint). `mailSend` is one function so the provider can be swapped.

### 7.2 Environment variables the user must set (Cloudflare Pages, Production and Preview; nothing in the repo)

| Name | Secret? | Purpose | If missing |
|---|---|---|---|
| `RESEND_API_KEY` | secret | Resend API key (sending-only scope) | `sendEmails` returns 503 `Email provider is not configured`; rows stay `not_sent` |
| `REDEEM_MAIL_FROM` | plain | Verified sender, for example `Cyndi Global <redeem@cyndiglobal.com>`. The domain must be verified in Resend (DNS records added by the user) | same 503 |
| `REDEEM_MAIL_REPLY_TO` | plain, optional | Support address | omitted |
| `REDEEM_PUBLIC_BASE_URL` | plain | Origin used in links, for example `https://www.cyndiglobal.com` | falls back to the request origin (authenticated request, so safe) |
| `REDEEM_MAIL_DAILY_LIMIT` | plain, optional | Invites per client per day | default 500 |
| `TURNSTILE_SITE_KEY` | plain | Public widget key (returned by `open`) | an open portal cannot be activated (409) and `open`/`submit` return 503 |
| `TURNSTILE_SECRET_KEY` | secret | Server verification | same |
| `REDEEM_MAIL_MODE` | plain, optional | `log` writes the message (recipient domain only, no token) to the log instead of sending. Local and demo only. Unset in production | n/a |
| `CUSTOMER_SESSION_SECRET` | secret, already set | Also derives the IP-hash key (`HMAC(secret, 'redeem-ip')`) | public endpoints 503 |

Local dev: values go in `.dev.vars` (git-ignored). Cloudflare publishes always-pass Turnstile test keys; look them up in the Turnstile docs.

### 7.3 Invite email

- Subject: portal `email_subject` or default "Your <portal title> link". CR/LF stripped.
- Body (HTML and plain text): greeting with the recipient name, `email_intro` (escaped, plain text), a button and the plain URL `/r/<token>`, and the line "This link is personal to you. Please don't forward it." All dynamic values are HTML-escaped.
- Recipient addresses come only from `redemption_codes` (admin-imported). Public input never becomes an email recipient.
- The senders are now client admins, who are outside Cyndi. Controls: the fixed template (the admin writes only a subject and a plain-text intro), `REDEEM_MAIL_DAILY_LIMIT` per client counted in `redemption_throttle` (`mail-day:c<clientId>`), 500 recipients per import, 1000 per portal, and `sendTest` limited to 5 per hour per admin (`test-mail:u<customerId>`). A client at its limit gets 429 with the reset time.

### 7.4 Send state machine and limits

`not_sent | failed → sending → sent | failed`. A row in `sending` with `sending_at` older than 10 minutes is treated as `failed` everywhere.

One `POST /emails` per invite, 10 invites per request, one after another (≤10 provider calls plus ≤12 D1 calls, within Pages subrequest limits). The UI repeats until `remaining = 0` and shows progress.

1. Claim: `UPDATE redemption_codes SET email_status='sending', sending_at=?now, send_count=send_count+1 WHERE client_id = ?c AND portal_id = ?p AND id IN (<ids>) AND status='active' AND (email_status IN ('not_sent','failed') OR (email_status='sending' AND sending_at < ?cutoff))`, then `SELECT` rows with `sending_at = ?now`. Rows another call won are reported as `skipped`. Without `codeIds`, the ids are the next 10 eligible rows by id. The daily limit is checked and incremented for the number of claimed rows before sending.
2. Send each row with `Idempotency-Key: rdm-<codeId>-<send_count>`. A retry of the same attempt reuses the key. An admin Resend increments `send_count`, so it is a new email.
3. Result per row: success → `sent`, `email_sent_at`, `email_error = NULL`; failure → `failed`, `email_error` (≤200 chars, no key).
4. Network error or timeout (8 s `AbortSignal.timeout`) marks that row `failed` "Provider unavailable" and continues. After 3 consecutive failures the loop stops and sets the rest back to `failed` in one UPDATE.
5. A crash after the provider accepted the mail but before the D1 write leaves the row in `sending`. After 10 minutes it can be resent with a new key, so one duplicate invite is possible. Accepted, rare, same link.

## 8. API and shared code

### 8.1 Endpoints

| Route | Auth | Methods |
|---|---|---|
| `/api/redeem` | none (public, throttled) | POST, GET `?media=` |
| `/r/<token>` | none | GET |
| `/api/showcase-admin/me` | Showcase admin | GET: `{admin:true, client:{code,name}, user:{displayName,email}, modules:['orders','inventory','redemption']}` |
| `/api/showcase-admin/orders` | Showcase admin | GET list and detail, POST op `cancel` |
| `/api/showcase-admin/inventory` | Showcase admin | GET only (POST returns 405) |
| `/api/showcase-admin/redemption` | Showcase admin | GET views and exports, POST ops |
| `/api/admin/redemption` | Cyndi admin | GET only, read-only global view, optional `?clientId=`. POST returns 405. Personal tokens and links are not returned. |
| `/api/showcase/auth` GET | customer session | existing; adds `showcaseAdmin: boolean` (cosmetic, used only to show the Admin button) |

### 8.2 Shared libraries (no duplicated business logic)

Moving logic out of the route files is the main refactor in this change. It is done first, with a before/after comparison (section 14.3, step 0), because Cyndi admin keeps using the same code.

| New module | Moved from | Exports | Scope parameter |
|---|---|---|---|
| `functions/_lib/orders-admin.js` | `admin/orders.js` (list query, `DETAIL_SQL`, `rowToDetail`, filters, status-change flow lines 336-410) | `ordList(db, params, scope)`, `ordDetail(db, no, scope)`, `ordChangeStatus(env, {orderNo, to, expected, actor, scope, allowedTo})` | `scope.clientId` null = all clients (Cyndi) |
| `functions/_lib/inventory-admin.js` | `admin/inventory.js` read functions (`listProducts`, `productDetail`, `loadOpenOrders`, `rowOut`, `movementOut`) | same names with a `scope` argument | `p.client_id = ?` joins |
| `functions/_lib/redemption-admin.js` | new | `rdmAdminGet(env, params, scope, opts)`, `rdmAdminPost(env, body, scope)` | required for every write |
| `functions/_lib/showcase-admin-auth.js` | new | `requireShowcaseAdmin` (2.1) | |

`admin/inventory.js` keeps its POST ops (`opTrack`, `opSet`, `opAdjust`, `opThreshold`) because only Cyndi staff write inventory. `admin/orders.js` becomes a thin route that calls `orders-admin.js` with `scope = {clientId: null}` plus the Cyndi-only ops (`deleteCancelledOrders`, `migrateShowcaseOrderNumbers`).

Detail query change: `LEFT JOIN clients cl ON cl.id = COALESCE(o.client_id, c.client_id)` so redemption orders show their client.

`ordChangeStatus` keeps the guarded batch exactly as today (release/ship/unship statements, order UPDATE last as the lock). With `scope.clientId` set, the lookup becomes `WHERE o.order_no = ?1 AND o.client_id = ?2` and the guard SQL in the stock statements adds `AND o.client_id = ?`. The Showcase admin route passes `allowedTo: ['cancelled']`, so every other target status returns 400 (AC21).

### 8.3 Showcase admin: orders

- List: same filters as the Cyndi list (status, search, paging) with `o.client_id = scope.clientId`. Includes `RDM-` and Showcase orders of the client; never `SW-` orders (their `client_id` is NULL).
- Detail: order lines, status, placed-by, and for `RDM-` orders `shipTo` and approval from `redemption_claims`. Shipment tracking is read through the existing shipment read helpers after the scope check (read-only).
- Cancel: from `pending` or `processing`, releases stock through the existing flow. Cancelling an `RDM-` order also runs the `counts = 0` statement (8.5).

### 8.4 Showcase admin: inventory (read-only)

`GET ?q=` product list with tracked variant counts, and `GET ?productId=` detail with variants, movements and open orders. Product list: `FROM showcase_products p WHERE p.client_id = ?`. Detail: refuses a product of another client with 404. `GET ?view=skus&q=` returns the flat stocked-SKU list through `invListStockSkus` with the caller's client (8.7); it is dropped if the provisional commits are removed. Stock write ops do not exist in this namespace.

### 8.5 Changes to existing code

| File | Change |
|---|---|
| `functions/_lib/order-status.js` | Add `awaiting_approval` to `ORDER_STATUSES`, label "Awaiting approval", `TRANSITIONS.awaiting_approval = []`. `transitionError` returns "Approve or reject this order on the Redemption page" for it. |
| `functions/_lib/orders.js` | `prepareInsertOrder` takes optional `status` (default `'pending'`) and `clientId`. Add `RDM_ORDER_NO_RE`, `isRedemptionOrderNo`, `isStockOrderNo` (showcase or redemption). Add one exported `isKnownOrderNo(no)` = `SW_ORDER_NO_RE` or Showcase or `RDM_ORDER_NO_RE`. |
| `functions/api/admin/orders.js` | **Delete the local `SW_ORDER_NO_RE`/`isValidOrderNo` (lines 22-23) and use `isKnownOrderNo` at lines 136 and 342** (review finding 1). `orderType()` returns `'RDM'` for `RDM-`; the type filter accepts `RDM`; the stock-effect gate uses `isStockOrderNo`; `deleteOrder` refuses `RDM-` orders with 409 "Redemption orders are kept for audit"; detail adds `shipTo` and `redemption`; list gets an optional `clientId` filter. `splitNotes` only strips `[showcase:<user>]`. The `[redeem:<portalId>]` prefix in `notes` is stripped by a new branch: the portal id goes to a `portalId` field of the detail and the drawer shows "Portal: <name>"; the remainder is the delivery note. |
| `functions/api/admin/shipments.js` | Replace the local `isValidOrderNo` (line 24) with `isKnownOrderNo`. |
| `functions/api/admin/shipments/upload.js` | Same (line 16), so photos for `RDM-` orders are accepted. `orders/shipment-photo.js` and `orders/mine.js` stay unchanged. |
| `functions/api/orders.js` | No change. A test asserts `RDM-` returns 404. |
| `functions/api/showcase.js` | Catalog and stock queries filter by the customer's `client_id`: categories and products `WHERE client_id = ?1`; stock `JOIN showcase_products p ON p.id = i.product_id WHERE p.client_id = ?1`; meta from `showcase_meta_client`. A customer with `client_id` NULL gets 403 "Your account is not linked to a client". `seedIfEmpty` writes `client_id` from `clients WHERE code='HRT'` (skipped when HRT is missing). Runtime DDL strings stay (the columns come from 0014). |
| `functions/api/showcase/orders.js` | Product lookup (line ~292) adds `AND p.client_id = ?` and `c.client_id = ?`. The order insert sets `client_id = customer.client_id`. |
| `functions/api/showcase/auth.js` | GET adds `showcaseAdmin` (one extra `client_admins` lookup). |
| `functions/api/admin/showcase.js` | Every op takes a required `clientId`; GET takes `?clientId=`. Category and product writes check the client exists and set `client_id`. **`saveProduct`** (the real upsert, `ON CONFLICT(id) DO UPDATE SET ... updated_at=?22${lock}`, `lock` = `WHERE showcase_products.colors_json IS ?23 AND showcase_products.sizes_json IS ?24 [AND <fingerprint> IS ?25]`): (1) the existing `existing` lookup also selects `client_id`. If `existing.client_id` is not NULL and differs from the requested client, return 409 "Product id belongs to another client" before the stock guard and before any write. (2) `client_id` becomes `?26` in the VALUES list and the INSERT column list. (3) For an existing row the `lock` gets one more term: `AND (showcase_products.client_id IS ?26 OR showcase_products.client_id IS NULL)`. A NULL row (an unbackfilled straggler) is therefore still editable and is not reported as `CHANGED_WHILE_SAVING`; a different client's row fails the term, the UPDATE changes 0 rows and the existing 409 path runs. (4) `client_id` is never in the `DO UPDATE SET` list except `client_id=COALESCE(showcase_products.client_id, ?26)`, which fills a NULL and never changes a set value. The category of a product must have the same `client_id`. Deletes add `AND client_id = ?`. Meta reads and writes use `showcase_meta_client`. |
| `functions/api/showcase/media.js` | **No change, accepted.** `showcase_media` has no `client_id`. The endpoint needs a Showcase customer session and the ids are `crypto.randomUUID()`, so another client's image is reachable only with a leaked id. The image of a client is not secret in the way orders or stock are. If the user wants it closed, the fix is `AND EXISTS (SELECT 1 FROM showcase_products p WHERE p.client_id = ?2 AND (p.image = ?3 OR <colour json_each match as in 6.2a>))` with the caller's client id. Recorded under decision 23. |
| `functions/api/admin/inventory.js` | Product list and the flat `skus` list take an optional `clientId` filter and return each product's client code. The list code moves to `inventory-admin.js` (8.7). `resetFromOrder` gets a required `clientId`, a client-scoped wipe and a reservation guard (8.7). The other POST ops are unchanged. |
| `functions/api/admin/customers.js` | New op `setShowcaseAdmin {customerId, enabled}`: requires an active customer with `showcase_access = 1` and a `client_id`; inserts or deletes the `client_admins` row; logs `admin customers: <user> set showcase admin <id> <on/off>`. Changing a customer's `client_id` or disabling the account deletes their row in the same batch. The customer list returns `showcaseAdmin: boolean`. |
| `product-showcase/public/admin/customers.html` | "Showcase admin" checkbox on a customer. |
| `product-showcase/public/admin/showcase.html` | Client selector at the top (uses `?resource=clients`), passed to every call. |
| `product-showcase/public/admin/orders.html`, `inventory.html` | Client column and filter. Orders: type filter `Redemption`, ship-to block in the drawer, "Awaiting approval" label, "No charge" for redemption totals. |
| `product-showcase/public/admin/*.html` (7 files: customers, index, inventory, logistics, orders, products, showcase) | Add a nav link "Redemption" and a card on the admin home. The new `redemption.html` (8th file) is the read-only page (10.2). |
| `tests/inventory.test.mjs` | Add a case for `prepareInsertOrder` with `status` and `clientId`. |

Orders-page cancel of a redemption order (Cyndi page or Showcase admin). Orders in `pending` or `processing` can reach the cancel path (`TRANSITIONS.processing` includes `cancelled`). An `awaiting_approval` order has no transitions, so Type B orders are decided only on the Redemption page, and an approved-then-cancelled claim keeps `approval_status = 'approved'`. The batch is, in order: the three `invTransitionStatements` with `effect:'release'` (existing guard `EXISTS (SELECT 1 FROM orders o WHERE o.order_no = ?1 AND o.status = ?4)`, `guardBinds:[from]`), then
`UPDATE redemption_claims SET counts = 0 WHERE order_no = ?1 AND counts = 1 AND EXISTS (SELECT 1 FROM orders o WHERE o.order_no = ?1 AND o.status = ?2)` with binds `(no, from)`, then the order UPDATE last (its `meta.changes` stays the optimistic lock). All statements use the pre-change status, so if another admin already changed the order, none of them take effect and the existing 409 path runs.

### 8.6 Redemption ops (`/api/showcase-admin/redemption`)

Same conventions as `admin/inventory.js`: auth before any DB access, parameterized SQL, `no-store`, JSON-only POST. Body caps: 8 KB for ops, 128 KB for `importRecipients`. Ids are validated by regex (`^rp_[a-z0-9]{10}$`, positive integers for code and claim ids) before any query. `rdmClassifyError(e) === 'missing_schema'` returns 503 `Run migration 0014_client_scope.sql and 0015_redemption.sql first`.

**GET**

| Request | Returns |
|---|---|
| `?` | Portal list of the client: id, name, type, status, counts (recipients, emailed, redeemed, pending approval, live orders). Pending-approval total for the badge |
| `?portalId=` | Portal fields, items with names/images and tracked status, and for `open` the full link |
| `?portalId=&view=recipients&page=` | Codes, 200 per page, with `link`, `emailStatus`, `opened`, `redeemed`, `orderNo` |
| `?portalId=&view=claims&status=&page=` | Claims, 100 per page: recipient, address, items, approval state, reviewer, order status, `dupHints` (live claims in the same portal with the same `ip_hash` or same lower-cased `line1` + `postal_code`) |
| `?products=1&q=` | The client's Showcase products for the item picker (id, name, image, tracked variants vs total) |
| `?export=recipients|claims&portalId=` | CSV, `Content-Disposition: attachment`, ≤ 5000 rows, every cell through `rdmCsvCell` |

**POST**

| op | Input | Behaviour |
|---|---|---|
| `createPortal` | type, name, title, texts, items[{productId,maxQty|null}], countries?, maxTotalOrders?, captchaRequired? | Validates (every product must belong to `scope.clientId`), generates id (+ `open_token` for open), status `draft`. Open portals force `requires_approval=1` and `captcha_required=1`. `captchaRequired` is not an input and is ignored if sent |
| `updatePortal` | portalId + same fields | Any status. Lowering a cap does not touch existing orders |
| `setStatus` | portalId, status | `active` requires ≥1 item, every variant of every item tracked (else 409 listing the products), for personal portals ≥1 active recipient, and for open portals `TURNSTILE_SITE_KEY` and `TURNSTILE_SECRET_KEY` set (else 409 `Captcha is not configured`; skipped only when `REDEEM_MAIL_MODE=log`). `paused`/`closed` always allowed. Closed can be re-activated |
| `deletePortal` | portalId | Only if no claims exist |
| `rotateToken` | portalId or codeId | New token; the old link stops working at once. For a personal code it also sets `email_status='not_sent'`, `email_sent_at=NULL`, `first_opened_at=NULL` and `sending_at=NULL`, because the emailed link is dead; the UI then offers "Send" again. For an open portal it changes `open_token` only |
| `importRecipients` | portalId, rows [{name,email}] ≤ 500 | Per-row validation as in 6.3, skips duplicates and bad rows, ≤ 1000 recipients per portal. Voided rows count toward the 1000, and re-importing a voided email is skipped as a duplicate, because the `(portal_id, email_norm)` unique index keeps the row. Returns `{created, skipped:[{row,reason}]}`. Tokens generated server side (16 random bytes) |
| `voidCode` | codeId | `status='void'`; the link shows UNAVAILABLE. Existing orders untouched |
| `sendEmails` | portalId, codeIds? (≤10) | Section 7.4 |
| `sendTest` | portalId, to | One sample invite (fake link) to one address; 5 per hour per admin |
| `approveClaim` | claimId, expectedApproval:'pending' | Batch: claim UPDATE guarded on the order still being `awaiting_approval`, then order UPDATE → `pending` last. Sets `reviewed_by_id`, `reviewed_by`, `reviewed_at` |
| `rejectClaim` | claimId, note (required, ≤500) | Batch: `invTransitionStatements(release)` guarded on `awaiting_approval`, claim UPDATE (`rejected`, `counts=0`, note), order UPDATE → `cancelled` last. 409 if someone else decided |

Failure handling: validation errors are 400 with a field message (not logged); stale state is 409 with the fresh row; an id outside the caller's client is 404; `inconsistent`/`insufficient` stock errors on release are 500 with "Inventory is inconsistent… contact Cyndi support", logged at `error`. Actions are logged as `showcase-admin redemption: <customerId> <op> <id>`. Recipient PII, tokens and emails are never logged.

### 8.7 The three provisional inventory commits (c3c3943, 28a7a6f, 44f6370)

Status is undecided: the user is asking the author whether to keep them. Rule for this design: **the redemption feature never calls and never reads `resetFromOrder`, `ivrResetFromOrder`, `ivwListStockSkus` or the `skus` list.** Everything below is a change to those functions only. If the commits are removed, delete this section's changes with them and nothing in sections 5 to 9 changes. If they are kept, the changes below are required before the first client-2 row exists, and AC27 covers them.

What the code does today (read from `functions/api/admin/inventory.js` at HEAD): `ivrResetFromOrder` (line 571) runs three unconditional statements in one `db.batch`: `DELETE FROM order_stock_lines`, `DELETE FROM inventory`, `DELETE FROM inventory_movements`, then seeds one tracked variant per line of one `SHOWCASE-yyyymmdd-nnn` order. `ivwListStockSkus` (line 206) selects every `inventory` row `LEFT JOIN showcase_products`, with no client filter, and returns it as `skus` in the GET list response.

| # | Conflict | Recommendation |
|---|---|---|
| 1 | The unconditional DELETEs wipe every client's stock, movements and reservations. `ivwListStockSkus` leaks other clients' stock to a client filter. | **Scope both by client; keep the reset in Cyndi admin only.** Details below. |
| 2 | Both functions live in the route file. | Move the list logic to `functions/_lib/inventory-admin.js` as `invListStockSkus(db, q, scope)`. The route file calls it. The Showcase admin route exposes it as `GET /api/showcase-admin/inventory?view=skus` (read only). The reset stays in the Cyndi route file (it is a Cyndi-only write) but its SQL helpers move to the lib as `invResetStatements(...)` so they can be unit-tested with a fake `db`. |
| 3 | A reset deletes in-flight `order_stock_lines` of pending redemption orders and breaks the reserve/release invariant. | **Refuse while the client has any open reservation owned by another order.** Details below. |
| 4 | Redemption item pool must not come from the stocked-SKU list (untracked products have no `inventory` row and are hidden there). | The pool comes from the catalog: `?products=1` in 8.6 reads `showcase_products WHERE client_id = ?`, and tracked-vs-total variants are computed by `invLoadForProducts` against the product matrix (`invProductMatrix`). Untracked products are listed with a warning chip, not hidden. The `skus` list is not used anywhere in redemption. |
| 5 | Migration 0014 and inventory keys. | 0014 adds **no** column to `inventory` (decision 12, section 5.4). The key stays `sku` PK + `UNIQUE (product_id, color_id, size)`. The list of queries in the new code that still change is below. |
| 6 | `IVR_ORDER_RE` only accepts `SHOWCASE-`. | Correct as is and left untouched. A `RDM-` order is a zero-price draw on stock, not an opening-stock count, so it is never a valid seed. `IVR_ORDER_RE` is deliberately narrower than `isKnownOrderNo` and stays local to the reset. `isKnownOrderNo` remains the only validator everywhere else (8.5). A unit test asserts `IVR_ORDER_RE` rejects `RDM-20261111-ABC123`. |

**`invListStockSkus(db, q, scope)`** (conflict 1):
- `scope.clientId` set (Showcase admin, or Cyndi with a client filter): `FROM inventory i JOIN showcase_products p ON p.id = i.product_id AND p.client_id = ?`. It is an inner join, so rows of deleted products (no product row, so no client) are never returned to a client.
- `scope.clientId` null (Cyndi, no filter): the current `LEFT JOIN` query unchanged, plus `p.client_id` selected so the page can show a client column. This is the only path that shows deleted-product orphans.
- The search term `q` filter is applied in JS as today; the page still hides products with no `inventory` row because that is how the page is built, and the redemption UI does not use it.

**`ivrResetFromOrder` changes** (conflicts 1 and 3):
1. Input: `clientId` is required (positive integer, must exist in `clients`). Cyndi admin only. No Showcase admin route can reach it (`/api/showcase-admin/inventory` rejects POST with 405). When the Cyndi roles ship, this op is `owner` only. The confirm string stays `RESET-INVENTORY`.
2. The seed order must satisfy `orders.client_id = clientId` (else 400 "Order belongs to another client"). Every product in its lines must have `showcase_products.client_id = clientId` (checked in the existing per-product loop; else 400). This also stops a mistyped order number from seeding another client.
3. **Scoped wipe.** Let `CS` = `SELECT i.sku FROM inventory i JOIN showcase_products p ON p.id = i.product_id WHERE p.client_id = ?1`. The three DELETEs become, in this order inside the same batch:
   - `DELETE FROM order_stock_lines WHERE sku IN (CS)`
   - `DELETE FROM inventory_movements WHERE sku IN (CS)`
   - `DELETE FROM inventory WHERE sku IN (CS)` (last, because `CS` reads `inventory`)
   Orphans (no product row) and other clients' rows are not in `CS` and survive. The result object still reports the three delete counts.
4. **SKU collision.** The current code starts `taken` empty because everything is wiped. With a scoped wipe, `taken` is preloaded with `SELECT sku FROM inventory WHERE sku NOT IN (CS)` before `invSkuCode` runs, so a generated code can never equal another client's code (that would violate the `sku` primary key and abort the batch with a 500).
5. **Open-reservation guard** (conflict 3). Before the batch the code runs
   `SELECT COUNT(*) FROM order_stock_lines l JOIN inventory i ON i.sku = l.sku JOIN showcase_products p ON p.id = i.product_id WHERE p.client_id = ?1 AND l.state = 'reserved' AND l.order_no <> ?2`
   (`?2` = the seed order). If the count is above 0 the op returns 409 "N open reservations from other orders (for example a redemption order awaiting approval). Approve, reject or cancel them first." with the first five order numbers. The seed order's own lines are deleted as today (a seed order is the order the stock is rebuilt from, so its reservation was against the old count); the response says so.
   The pre-check can race with a submit that arrives before the batch commits. To make the check atomic, the batch starts with a **tripwire** statement: `INSERT INTO inventory_movements (sku, type, actor, created_at) SELECT NULL, 'guard', 'system', ?3 WHERE EXISTS (<same open-reservation query>)`. When an open reservation exists, this inserts a NULL into the NOT NULL column `sku`, the statement fails, and D1 rolls the whole batch back. The route maps that error to the same 409. Build step: provoke it once locally and confirm the batch leaves the data unchanged.
   Shipped and released `order_stock_lines` of the client are deleted as today. This loses the shipped history for an `unship` of an old order, which is an existing property of the reset and is stated in the confirmation text.
6. Order-number check: the existing `IVR_ORDER_RE` stays (conflict 6).

**Queries in the new code that touch inventory keys or client scope (conflict 5), complete list:**
- `ivwListStockSkus` SELECT (join + filter, above).
- `ivrResetFromOrder`: the order SELECT (adds `client_id`), `loadProduct` (the product must carry `client_id`; one more selected column), the three DELETEs (scoped), the `taken` preload (new SELECT), the seed `INSERT INTO inventory ... SELECT ... FROM n` (column list unchanged because 0014 adds none), the seed `INSERT INTO inventory_movements` (unchanged).
- The `UPDATE showcase_products SET colors_json = ...` backfill statements get `AND client_id = ?` (belt and braces; they already use the old `colors_json` as an optimistic lock).
- If the user chooses the alternative in decision 12 (a `client_id` column on `inventory`), the seed INSERT gains `client_id`, `CS` becomes `SELECT sku FROM inventory WHERE client_id = ?1`, and `ivwListStockSkus` filters on `i.client_id`.

Failure behaviour of the reset: validation errors are 400 with a message, stale or blocked state is 409, an unexpected D1 error is 500 `Could not reset stock from order` and is logged with the message only (no PII). Every error leaves the data untouched, because the batch is one transaction and all lookups run before it. The reset is logged: `admin inventory: <user> reset client <code> stock from <order>: <n> rows / <u> units`.

## 9. Admin UI

### 9.1 Showcase admin (the `showcase/` repo, React)

- `ShowcaseHeader` shows an "Admin" button when `/api/showcase/auth` returns `showcaseAdmin: true`. The server stays the authority: every admin call re-checks (2.1).
- New view `admin` in `App.jsx` (the app switches on a `view` state, no router). Components under `showcase/src/components/admin/`: `AdminShell` (tabs Orders / Inventory / Redemption, client name from `/me`), `AdminOrders`, `AdminInventory`, `AdminRedemption`, helpers in `showcase/src/data/adminApi.js`.
- **Orders**: filterable list, detail drawer with ship-to for redemption orders, Cancel button only while `pending` or `processing`.
- **Inventory**: read-only table of the client's products with available / reserved / incoming per variant and recent movements.
- **Redemption**:
  1. Portal list: name, type, status, counts, pending badge, "New portal".
  2. Portal editor: type (locked after create), internal name, public title, intro, success text, email subject and intro (Type A), country restriction, max total orders (Type B; captcha is always on and shown as a read-only note), item picker with search, per-item cap field (empty = unlimited) and a "set all to 1" shortcut, warning chips for untracked variants ("ask Cyndi to track stock"). Activate, Pause, Close. A preview of the public wording.
  3. Recipients tab (Type A): paste box "Name, email" per line or a `.csv` parsed in the browser; summary of skipped rows; table with name, email, email status, opened, redeemed, Copy link, Resend, Void; "Send all unsent" (loops 10 at a time with progress), "Send test email to me", "Export CSV (with links)".
  4. Link panel (Type B): the open link, Copy, "Rotate link" (confirm), usage counts.
  5. Claims tab: filter (pending approval default for Type B), rows with recipient, country, items, dup hints, time; detail drawer with the full address; Approve and Reject (reject needs a note; both confirm). Export CSV with a PII warning.
  6. A short help text explaining the two types in plain English.
- All dynamic text via React (escaped by default); no `dangerouslySetInnerHTML`. Buttons disable while a request runs. Errors show the API message. CSV is fetched and saved from a blob.
- Deploy: the app is built into `/showcase/` by `scripts/build_showcase.mjs`. Separate repo, separate commit. The Admin button is hidden unless the API says so, so either repo can be deployed first.

### 9.2 Cyndi admin (main site, vanilla HTML)

- `redemption.html`: read-only list of all portals across clients (client column, filter by client), counts, claims list, CSV export. No tokens or links shown. Pending Type B orders show "Awaiting approval, decided by the client".
- Customers page: "Showcase admin" checkbox (8.5). Showcase page: client selector. Orders and Inventory pages: client filter.
- Fulfilment: `pending` redemption orders appear on the Orders page like any Showcase order and ship through the existing Shipments tab.

## 10. Security

Scope: this adds a public unauthenticated surface that writes orders, reserves stock, stores PII and triggers email, plus a new class of authenticated external user (client admins) who can send email and read order data. Treated as high risk. The builder-security skill was loaded and applied: the public endpoints are legitimately unauthenticated (recipients are outside people), so no login is added to them; every admin route keeps its session gate; no bind or listener changes (local dev stays on `127.0.0.1`). **The ARCC query (`search_arcc`) failed twice with a Midway 401 ("run mwinit"). No ARCC guidance was retrieved. Re-run it after `mwinit` and treat this section as provisional until then.**

### 10.1 Threats and controls

| Threat | Control |
|---|---|
| Client A admin reads or changes client B data | `client_id` from the DB row only (2.1), scope helpers return null, 404, composite FKs, static scope test, cross-client test matrix (14.3) |
| Customer cookie used on Cyndi admin routes, or the reverse | Different `aud` values, tested both ways (AC2) |
| A stale admin keeps access after being removed | Membership, customer status and client status are read on every request (AC3) |
| CSRF on Showcase admin (cookie is `SameSite=Lax`) | JSON-only POST, Origin and `Sec-Fetch-Site` checks, no CORS, state changes only via POST |
| Client admin abuses email (spam, phishing) | Fixed template, plain-text intro, per-client daily cap, per-portal and per-import caps, test-mail limit, recipients only from admin-imported rows |
| Guessing a personal link | 128-bit token, unique index, failed-lookup throttle |
| Open link leaks | Admin approval on every order (DB-enforced), Turnstile, unique email, soft cap, pause and rotate |
| Stock exhaustion by flooding pending orders | Same controls, throttle, and reject releases stock |
| Free items through untracked variants | Activation check and submit refusal when `stockSku` is null |
| Mail relay | No mail to public input in v1 |
| Enumeration of other recipients | `open` returns only the token holder's display name; unknown, voided and wrong tokens look the same except paused vs unknown, seen only by a token holder |
| Stock number leakage | Only boolean `inStock`; no counts in errors |
| XSS via names, addresses, portal text | Stored raw, rendered with `textContent` / React escaping; admin pages keep `escapeHtml`; email HTML-escaped; CSP without inline script |
| CSV injection | `rdmCsvCell` prefixes `'` for `= + - @ \t \r` and quotes cells |
| SQL injection | All SQL parameterized; interpolated parts are fixed literals and generated `?N` placeholders |
| Double submit / replay | `idem_key` unique index |
| Race on one link | Partial unique indexes and the stock CHECK, not app-level checks |
| Token leakage via logs/referrer | Token in POST body; `Referrer-Policy: no-referrer`; logs never contain tokens, emails or addresses |
| Secrets | Only Cloudflare env secrets; nothing in the repo |
| Cross-client product in a portal | App check on create, update, activate and submit; the public `open` query filters on the portal's client |
| Cross-client overwrite through `saveProduct` upsert | Conflict clause requires equal `client_id` (8.5) |

### 10.2 Throttling

Key = `<action>:<HMAC-SHA256(CF-Connecting-IP)>` (key from `CUSTOMER_SESSION_SECRET`). Fixed windows. `rdmThrottleHit(db, key, windowSecs, limit)` is one `INSERT ... ON CONFLICT(key, window_start) DO UPDATE SET count = count + 1 RETURNING count`; allowed when the count ≤ limit. `rdmThrottlePeek` is a `SELECT count` only, used for `open-fail`.

When `CF-Connecting-IP` is absent, the IP part is `local` only if `env.REDEEM_MAIL_MODE === 'log'` or the host is `127.0.0.1`/`localhost`; otherwise public endpoints return 503, so a dropped header cannot put all traffic in one bucket.

| Action | Limit | When counted |
|---|---|---|
| `open-fail` | 10 per 10 min per IP | Peek at the start of `open` and `submit`. Hit only on unknown or malformed tokens and honeypot hits |
| `submit` | 20 per hour per IP | Once per submit after the token resolves, before captcha and body validation. Above 5 so a client office behind one NAT can redeem at an event |
| `portal-submit` | 60 per hour per open portal | After the captcha passed |
| `media` | 300 per hour per IP | Each `GET /api/redeem?media=` |
| `test-mail` | 5 per hour per admin | `sendTest` |
| `mail-day` | `REDEEM_MAIL_DAILY_LIMIT` per day per client | `sendEmails` |

If the throttle table is missing or D1 fails, public `submit` fails closed (503). `open` fails open for tokens that resolve; unknown tokens still count when the table works.

Defence in depth for the user to add by hand in the Cloudflare dashboard, if the account's plan allows a rate-limiting rule (check the Cloudflare rate-limiting docs and the plan limits first; this design did not verify the number of rules per plan): a rule on `/api/redeem` (for example 30 requests per minute per IP, Block). A D1 counter only stops casual abuse under a real flood.

### 10.3 Verification of denial and exposure (part of the test plan)

- Every Cyndi admin GET and every `/api/showcase-admin/*` method without a cookie, with a garbage cookie, with an expired cookie, with the wrong tier's cookie, and with a non-admin customer cookie: 401 or 403 and no body data.
- `/api/redeem` GET without `media` and OPTIONS: 405; `?media=` for an id outside active pools: 404; no `Access-Control-*` headers anywhere.
- `grep -rnE "0\.0\.0\.0|--host|listen\(|\[::\]"` over `functions/`, `product-showcase/`, `showcase/src`, `scripts/` and config after the build; wrangler is started with `--ip 127.0.0.1`; `lsof -nP -iTCP -sTCP:LISTEN | grep 8788` shows `127.0.0.1`.
- Public deploy check inverts: `/r/<unknown>` returns the shell (200), `/api/redeem` with an unknown token returns 404, and no admin or stock data is reachable anonymously.

### 10.4 PII

Stored: recipient name, email, phone, postal address, IP hash. Needed to ship. A Showcase admin sees the data of their own client's recipients only. Retention and a purge job are out of scope but listed in section 16. CSV exports contain PII; the UI warns before export.

## 11. Roles

- Showcase admin is a membership row in `client_admins`: all of Orders (view and cancel), Inventory (read-only) and Redemption (full) for that client. No finer roles in v1.
- Who may approve a Type B order: any Showcase admin of the client. Fei and Megan each have their own account, so `reviewed_by` shows who decided. Approving your own request is not blocked (a recipient is not an admin account, so the case does not arise).
- Cyndi `admin_users` has no `role` column and every Cyndi admin can do everything. The planned owner/staff/warehouse roles are not needed for this feature, and nothing here depends on them. When they ship, the Cyndi read-only Redemption page maps to all three roles, and "grant Showcase admin" (`setShowcaseAdmin`) maps to `owner` only. The old `rdmCan` seam is dropped: the membership check replaces it.
- The shared HRT storefront account (username `HRT`) is not a Showcase admin and must not become one: admins must be personal accounts. Granting is blocked for accounts whose email ends in `@clients.cyndiglobal.invalid`.
- Removing access: delete the `client_admins` row or disable the customer; effective on the next request.

## 12. Why the public page is in the main site

The Showcase app is login-gated and cloned into `/showcase/` at build time. A public no-login page there would need exceptions in its auth and a second CSP. The main site already serves static pages and Pages Functions, so `/r/<token>` and `/api/redeem` live there. The Showcase admin UI must live in `showcase/` because that is where the client admins sign in (addendum section 1).

## 13. Implementation order (for the next step)

0. Refactor with no behaviour change: extract `orders-admin.js`, `inventory-admin.js`, add `isKnownOrderNo`; capture before/after JSON from the Cyndi Orders and Inventory GET endpoints and compare.
1. Migrations 0014 and 0015, `_lib/redemption.js`, unit tests.
2. Order, status and shipment changes (8.5), including the `isKnownOrderNo` fix.
3. Client scoping of catalog, orders and Cyndi Showcase page (8.5), with tests.
4. `requireShowcaseAdmin`, `showcase-admin/*` routes for me, orders, inventory, `setShowcaseAdmin`.
5. Public endpoint, `/r/` function and page.
6. `redemption-admin.js` and the redemption route. In step 3 also apply 8.7 (client scope for the provisional inventory functions) if the commits are still kept at that time.
7. Check Resend docs (7.1), then the mailer.
8. Showcase admin UI in `showcase/`, Cyndi `redemption.html` and nav links.
9. Verification run (section 14).

## 14. Test plan

No Playwright. Never `wrangler d1 ... --remote`.

### 14.1 Unit tests: `node --test tests/` (style of `tests/inventory.test.mjs`, fake `db` recording statements)

`tests/redemption.test.mjs`:
- Token generator: length 22, charset, 1000 samples unique.
- Validators: each rule in 6.3 with accept and reject cases.
- Cap logic: sum across colours, unlimited = 100, product outside pool, missing colour/size, product of another client.
- Snapshot builder: no prices, `stockSku` set, `currency: null`.
- Public view builder: no counts, no `stockSku`, no client id, personal view has no email.
- `rdmClassifyError`: one case per message (idem_key, code_id, open-portal email, `inventory` missing, `order_stock_lines` missing, `redemption_claims` missing, `no such column: client_id`). Exact D1 text for the partial unique indexes comes from provoking each locally once.
- Statement builders: constant statement count; approve and reject guard SQL; cancel-claim statement; no unbound `?N`.
- Throttle: window math, key has no raw IP, `Hit` vs `Peek`, `open-fail` not counting successes, missing IP gives 503 outside log/local mode, `portal-submit` untouched when the captcha fails.
- CSV formula guard, quotes, newlines. Import parser: duplicates, bad emails, row limits, BOM, quoted commas.
- Mail builder: HTML escape, CR/LF stripped from subject, link from base URL, no token in log mode. Send claim: stale `sending` reclaimed after 10 minutes, fresh not; two overlapping claims never own the same row; idempotency key format; daily limit.
- `rdmPublicImage`: static path, R2 origin, media rewrite, foreign host, `javascript:`, bad id.
- Media authorization statement: run against an in-memory SQLite-compatible fixture if available, otherwise assert the SQL text has no `INSTR`/`LIKE` and uses both binds; the integration test 16 covers behaviour.
- Idempotency decision function (pure): `(found, counts)` gives replay 200, `KEY_USED` 409 or continue; the page's key lifecycle (delete after 200 / 409, keep after 5xx) as a small pure helper.
- Portal validation: `createPortal` for `open` ignores `captchaRequired`; `setStatus` activation rule with and without Turnstile env.
- `saveProduct` lock builder: SQL text and bind order for existing/new product, with and without `invRows`, NULL `client_id` allowed, other client refused.
- `invListStockSkus` and `invResetStatements` with a fake `db`: every statement carries the client bind, the DELETE order is lines, movements, inventory, the tripwire is first, `taken` preload present. `IVR_ORDER_RE` rejects `RDM-`.

Other unit tests:
- `isKnownOrderNo` accepts `SW-20261111-ABC123`, `SHOWCASE-20261111-001`, `SC-20261111-ABC123` and `RDM-20261111-ABC123`, rejects `RDM-1`, `RDM-20261111-abc123` and empty (review finding 1).
- `order-status`: `awaiting_approval` has no next; error text. `prepareInsertOrder` with `status` and `clientId`. `isStockOrderNo`.
- `showcase-admin-auth` with a fake `db`: 401 no session, 403 no access, 403 no client, 403 no membership, 403 membership for a different client than the customer's, 200 otherwise; the returned `clientId` is the customer's; `Origin` mismatch gives 403 on POST.
- Scope lint test: reads `functions/_lib/{orders-admin,inventory-admin,redemption-admin}.js` and `functions/api/showcase-admin/*.js`, extracts SQL string literals, and for every statement that mentions `orders`, `showcase_products`, `showcase_categories` or `redemption_` requires `client_id` unless the statement is in a small allowlist that sits in the test file with a reason per entry.

### 14.2 Lint and build

`cd product-showcase && npm run lint && npx vite build`; `cd showcase && npm run lint && npx vite build`; `node --test tests/`. Check that `dist/redeem.html`, `redeem.js`, `redeem.css` exist.

### 14.3 Local integration (curl)

Setup: apply 0014 and 0015 to the local DB (`npx wrangler d1 execute cyndi-logistics --local --file=migrations/...`). Start: `npx wrangler pages dev product-showcase/dist --port 8788 --ip 127.0.0.1` after `npx vite build`. Log in as a local Cyndi admin (ask the owner for the local test account); cookie jars with `-c/-b`. `REDEEM_MAIL_MODE=log` and Cloudflare's published Turnstile test keys in `.dev.vars`.

Fixture `tests/fixtures/two_clients.sql` (local only): client TESTB; customers `adminA` (HRT, Showcase admin), `plainA` (HRT, no admin), `adminB` (TESTB, Showcase admin); one product, tracked stock, one portal and one order for each client.

0. Refactor baseline: save JSON of Cyndi `GET /api/admin/orders` (list, detail, `type` filter) and `GET /api/admin/inventory` before the extraction and diff after. Must be identical.
1. Denial matrix: every Cyndi admin GET and every `/api/showcase-admin/*` method with no cookie, garbage cookie, expired cookie, wrong tier's cookie, `plainA` cookie: 401/403 and no data (AC1-AC3).
2. Cross-client matrix (AC4): logged in as `adminA`, call every read and write op on client B's portal, code, claim, order, product and SKU: 404, and the DB rows are unchanged afterwards (compare a checksum query). Send `clientId=<B>` in query and body: ignored.
3. Revocation: delete `adminA`'s `client_admins` row, disable the customer, disable client HRT, move the customer to TESTB: each gives 403 on the next call.
4. Catalog: `plainA` and `adminA` see only HRT products; a TESTB customer sees only TESTB's; a customer with `client_id` NULL gets 403. Place a Showcase order with a TESTB product id as an HRT customer: 400.
5. Cyndi `saveProduct` with an id owned by another client: 409. Same client: succeeds.
6. Type A: create a portal with 2 tracked items (cap 1 and unlimited); activation fails with an untracked variant, passes once tracked. Import 5 rows with 1 duplicate and 1 bad email. `sendEmails` with the provider unset: 503 and rows stay `not_sent`; with log mode: `sent`; a second call is a no-op. Set a row to `sending` with an old `sending_at` by SQL: picked up; fresh: not. Daily limit reached: 429.
7. Public `open` with a valid, unknown, voided, paused token and a token of a disabled client: 200 / 404 / 410 / 410 / 410. Unknown ×11: 429. Bodies contain no `available`, `reserved`, `stockSku`, other emails.
8. Submit happy path: check `orders` (status, `client_id`), `order_stock_lines`, `inventory_movements`, claim row.
9. Ten parallel submits on one link: exactly one 200, others 409, `reserved` up once. Same idempotency key twice: same order number.
10. Cap exceeded, wrong product, bad country, bad phone, honeypot: 400 and no rows written. Stock: more than available gives 409 without numbers; stock 1 with two parallel submits: one wins; `available` never negative.
11. Cancel the Type A order as `adminA` and again on the Cyndi Orders page (from `pending`, and from `processing` after a status change): stock released, `counts=0`, link usable again.
12. Type B: submit gives `awaiting_approval`; `processing` on the Orders page returns 400 (not "Invalid order number"); `adminA` approves → `pending`; Cyndi ships → stock deducted. A second submit rejected with a note: `cancelled`, `available` restored, `release` movement; same email can submit again; duplicate email while live: 409.
13. `max_total_orders` reached: 409 (a parallel burst may overshoot slightly, documented); pause: 410; rotate: old link 404.
14. CSV: a name starting with `=` is escaped.
15. `GET /api/orders?orderNo=RDM-...` is 404. `/api/admin/orders?orderNo=RDM-...` returns the detail. Shipments: add a shipment and upload a photo for an `RDM-` order: both succeed. Delete of a cancelled `RDM-` order is refused and the claim stays.
16. Images: a product whose image is `/api/showcase/media?id=...` appears as `/api/redeem?media=<id>` and returns 200 anonymously; an id outside active pools returns 404. Fetch every image URL in the response anonymously: none returns 401.
17. An item with all variants untracked after activation is omitted from `open`; 410 when it was the last.
18. `/r/<token>` returns 200 (not 308), shell body, CSP with `img-src` from `R2_PUBLIC_BASE`, `Referrer-Policy`, `no-store`. `/r/a/b` returns 404.
19. Showcase admin `POST /inventory` returns 405; `POST /orders` with `status: 'shipped'` returns 400; `POST` with `Origin: https://evil.example` returns 403; `POST` with `text/plain` returns 415.
20. Migration state: a fresh local DB without 0015: admin 503 naming the migration, public 503 generic. A DB with 0015 but without 0014's columns: catalog 503, not a 500 stack.
21. Backfill checks (local copy with seeded rows): after 0014, `SELECT COUNT(*) FROM showcase_products WHERE client_id IS NULL`, same for categories, and `... FROM orders WHERE client_id IS NULL AND (order_no LIKE 'SC-%' OR order_no LIKE 'SHOWCASE-%')` are all 0 (AC22). Re-run Section B twice: no change.
22. Exposure sweep from 10.3 and the `lsof` check.
23. Idempotency (review finding 1): submit on a Type A link, repeat with the same key: 200 and the same order number, no throttle increment. Submit with a new key: 409 `ALREADY_REDEEMED`. Cancel the order, resubmit with the old key: 409 `KEY_USED`; with a new key: 200 and a new order. Same sequence on Type B with reject.
24. Media (finding 2): an id equal to a pool product's media URL returns 200 anonymously. An id that is a prefix/substring of it, an image only in `gallery_json`, a product of a draft portal, and a disabled client's portal return 404. Colour image stored with a different key order still matches.
25. Captcha (finding 3): create an open portal; `captchaRequired:false` in the body is ignored and the row has 1. `UPDATE redemption_portals SET captcha_required = 0 WHERE type='open'` by SQL fails on the CHECK. Unset the Turnstile vars: `setStatus active` returns 409; with `REDEEM_MAIL_MODE=log` it passes. Submit with the keys unset: 503.
26. `saveProduct` (finding 4): an id owned by another client: 409, row unchanged. A row with `client_id` NULL: saved and filled. Same client with a stale `colors_json`: still the existing `CHANGED_WHILE_SAVING` 409.
27. `rotateToken` on a sent personal code: `email_status = 'not_sent'`, `email_sent_at` and `first_opened_at` NULL, old link 404.
28. Inventory reset (8.7): with two clients' stock and one pending Type B reservation of client A, `resetFromOrder` for A is refused with 409 and nothing changes; after rejecting the order it succeeds, A's rows are rebuilt and B's `inventory`, `inventory_movements` and `order_stock_lines` are byte-identical (compare checksum queries). Seed order of client B for `clientId=A`: 400. Concurrent submit during the reset: the tripwire rolls the batch back. `GET ?view=skus` as `adminA` lists only A's SKUs. Cyndi `GET` with `?clientId=` filters. A `RDM-` order number as seed: 400.
29. Showcase admin UI smoke by hand (no Playwright): sign in as `adminA`, create both portal types, import, send in log mode, approve, reject, export.

## 15. Migration plan, rollout, rollback, risks

### 15.1 Pre-flight (user pastes in D1 Studio Console, read-only)

```
SELECT id, email FROM customers WHERE showcase_access = 1 AND client_id IS NULL;   -- must return no rows, or assign a client first on the Customers page
SELECT code FROM clients WHERE code = 'HRT';                                        -- must return one row
```

### 15.2 Order

1. Pre-flight queries (15.1).
2. Paste `0014_client_scope.sql` Section A, one ALTER at a time (skip any that fail with "duplicate column name"), then Section B.
3. Verify: the three `client_id IS NULL` counts in test 21 return 0. The old code keeps working after step 2 because it ignores the new columns, but a product or Showcase order created by old code between steps 2 and 5 gets `client_id` NULL. Re-run the backfill `UPDATE`s of Section B just before step 5 (they are re-runnable).
4. Paste `0015_redemption.sql` (re-runnable).
5. Deploy the main repo (code from 8.5, 8.6 and the public page). Deploying before step 2 would break the catalog with "no such column", which is why the code returns a clear 503 for that case.
6. Set env vars (7.2) and the optional WAF rule.
7. On the Customers page create Fei's and Megan's own accounts (HRT client, Showcase access on), then tick "Showcase admin".
8. Deploy the `showcase/` repo (push `showcase` first, then the main site, per PROGRESS.md; either order works because the Admin button depends on the API).
9. Verify admin anonymous denial and public `/r/x` = 404 on the live site, then do one real test round with the client.

Nothing here runs `wrangler d1 ... --remote`; the user performs steps 2 and 4 by hand.

### 15.3 Rollback

- Code: redeploy the previous commit. The old code ignores `client_id`, `client_admins`, `showcase_meta_client` and the redemption tables, and `showcase_meta` was kept, so the old catalog works unchanged.
- Data: leave the columns and tables in place (harmless). Redemption orders that exist stay in `orders` with status `awaiting_approval`, which the old Orders page lists but cannot move; cancel or reject them before rolling back, or leave them.
- Full removal (only if wanted): `DROP INDEX` the three new indexes, then `ALTER TABLE ... DROP COLUMN client_id` on the three tables, `DROP TABLE` the redemption tables (children first) and `client_admins`, `showcase_meta_client`. Not done automatically.
- 0015 is safe to leave unused.

### 15.4 Risks

| Risk | Impact | Mitigation |
|---|---|---|
| `client_id` on `orders` and the catalog tables touches many queries (list in 8.5) | A missed query leaks or hides data across clients | Scope helpers, static scope test, cross-client matrix (test 2), baseline diff for the refactor (test 0) |
| Extracting `orders-admin.js` and `inventory-admin.js` changes code that works today | Regression on the Cyndi Orders and Inventory pages | Step 0 before any feature work; before/after JSON diff; no behaviour change allowed in that commit |
| `client_id` nullable at the database level | A code path that forgets it creates an invisible row | Every insert sets it; tests assert it; later table rebuild to NOT NULL |
| Product and category ids are global | Client B cannot reuse an id used by client A | Accepted. Cyndi admin gets a clear 409. A per-client id prefix can be added when client #2 starts |
| Product `client_id` immutable | Moving a product between clients is not possible | By design; `saveProduct` refuses a client change |
| Inventory scoped by join, not by column | A future query on `inventory` without the join would be unscoped | Static test covers the scoped modules; decision 12 offers the column alternative |
| Showcase customers without a client | They lose catalog access | Pre-flight query; clear 403 message |
| `showcase_meta` singleton | Per-client settings were not possible | Moved to `showcase_meta_client`; the old table is unused |
| Client admins can send email | Spam or phishing from the Cyndi domain | Fixed template, caps (7.3), plain-text intro |
| Customer cookie is `SameSite=Lax` | CSRF on Showcase admin POST | JSON-only plus Origin / Sec-Fetch-Site checks |
| Manual D1 steps | A skipped or misordered step | Section 15.2 with verification queries; code returns a clear 503 for missing schema |
| Two repos | UI and API deployed at different times | Admin button depends on the API flag; API never requires the UI |
| Stock held by unreviewed Type B orders | Stock stays reserved until a decision | Reject releases it; soft cap; no auto-expiry in v1 |
| Resend, Turnstile and D1 index error text unverified | Build-time surprises | Named build steps, not assumptions |
| Provisional inventory commits kept unscoped | One reset wipes all clients' stock; the `skus` list leaks across clients | 8.7: scoped wipe, reservation guard, owner-only, inner join by client. Redemption does not depend on them |
| A Type B order waiting for approval holds stock during a reset | Reserve/release invariant broken | Reset refused while another order holds a `reserved` line (atomic tripwire) |
| ARCC not consulted | Governance guidance missed | Re-run after `mwinit` before implementation |

## 16. Decisions needing user confirmation

Each has a recommendation. Reply with changes; otherwise these go into implementation.

| # | Decision | Recommendation | Why / alternative |
|---|---|---|---|
| 1 | Orders: reuse `orders` + `RDM-` prefix + `redemption_claims` | Reuse | Reservation, Orders page, Shipments and CSV all key on `orders`. A separate table needs a second ship/stock path. |
| 2 | Structured address lives only in `redemption_claims` for now | Yes; add a general `order_addresses` later | `ALTER TABLE orders` for addresses is another non-re-runnable change. |
| 3 | New order status `awaiting_approval` | Yes | The warehouse reads one field. A flag on `pending` would let staff ship an unapproved order. |
| 4 | Link `/r/<22-char token>`, personal tokens stored in plain text | Yes | Plain text is needed for resend and the links CSV. Alternative: store a hash and issue a new link on every resend. |
| 5 | Anti-abuse set: D1 throttle + Turnstile (open) + one live order per email + soft `max_total_orders` (200) + WAF rule by hand | Yes | You create a Turnstile widget and two env vars. Alternative: no Turnstile, rely on approval alone. |
| 6 | Item model: per-product cap summed over colours/sizes, "unlimited" = 100 per order, any colour/size | Yes | Alternative: per-variant pools if the client wants only some colours. |
| 7 | Submissions: Type A one live order per link, Type B one live order per email | Yes | Matches "1 of each item". Alternative: N orders per recipient (app-level counting only). |
| 8 | Email: Resend, invites only; per-client daily cap 500 | Yes | You create a Resend account, verify the sender domain DNS, set `RESEND_API_KEY` and `REDEEM_MAIL_FROM`. Receipts to public-typed addresses on Type B would open a spam route. |
| 9 | Approver notification email when a Type B order arrives | No in v1; badge in the Admin tab only | Megan and Fei may expect an email. Ask them. |
| 10 | Untracked variants cannot be redeemed; pending Type B orders hold stock until decided | Yes | Without tracking, redemption is unlimited. Auto-release after N days is a later option. |
| 11 | Public page in the main site; admin UI in `showcase/` | Yes | Section 12. |
| 12 | Inventory gets no `client_id` column; scoping is through the product | Yes | Avoids a copied value that can drift (5.4). Alternative: add the column to `inventory` (one more ALTER, one more invariant). |
| 13 | Showcase admin rights: Orders view + cancel, Inventory read-only, Redemption full | Yes | Fei asked to "see inventory and orders". Alternative: let client admins edit stock (more risk, Cyndi owns the warehouse count). |
| 14 | Cyndi admin gets a read-only Redemption page, no approve/reject | Yes | If both client approvers are away, Cyndi must ask the client. Alternative: allow Cyndi owners to decide on behalf of the client, with an audit note. |
| 15 | Allowed countries: ISO list minus (CU, IR, KP, SY, RU, BY) | Confirm the real shipping list with the client/forwarder | HRT v1 doc says SG, IN, HK, CN, AU; the latest message says "anywhere you can ship". |
| 16 | Per-IP submit limit 20/hour | Yes; confirm | An office NAT may need more for events. |
| 17 | PII retention and purge job | Out of scope now; decide a retention period | Needed before a large or non-US recipient list. |
| 18 | Client testing: use a small real portal and cancel test orders (releases stock). No sandbox mode | Yes | A sandbox flag adds a code path that only tests itself. "Send test email to me" and Draft preview are included. |
| 19 | Product and category ids stay globally unique across clients | Yes for now | A per-client prefix can come with client #2. |
| 20 | Showcase customers with no client are refused | Yes | The pre-flight query shows if any exist. |
| 21 | Fei and Megan use personal accounts, not the shared `HRT` account | Yes | Per-person approvals in the audit trail. |
| 22 | Migration split: 0014 (client scope, has 3 non-re-runnable ALTERs) and 0015 (redemption, re-runnable) | Yes | Keeps the re-runnable part separate. |
| 23 | `/api/showcase/media` stays unscoped (random UUID ids) | Accept for now | One extra `EXISTS` closes it if wanted (8.5). |
| 24 | Provisional inventory commits: keep, scoped as in 8.7 | Keep the three commits only if the author confirms; either way apply 8.7 or delete the code. Redemption works in both cases | The reset is a destructive Cyndi-owner tool. Alternative: remove `resetFromOrder` and keep only the SKU list. |
| 25 | Open portals always require captcha, no toggle; activation blocked until Turnstile keys exist | Yes | Alternative: allow an open portal with approval only. Rejected: the approval queue is the thing a bot flood would fill. |
| 26 | Reset guard: refuse while any other order holds an open reservation | Yes | Alternative: only protect redemption orders. Rejected: a wiped Showcase reservation corrupts stock the same way. |

## 17. Questions that change the work most

Decision 13 (client admins cannot edit stock), decision 8 (Resend account and sender-domain DNS), decision 12 (no `client_id` column on inventory), decision 15 (country list).

## 18. Review responses

### Current review (revision 4: CHANGES_REQUESTED, 3 MEDIUM, 5 NIT)

| # | Finding | Response |
|---|---|---|
| 1 | MEDIUM: idempotent replay vs live-claim check; reused key returns a cancelled order | Fixed. 6.4 step 3 looks up `(portal_id, idem_key)` before the live-claim check (step 4): `counts=1` returns 200 with the order number and no throttle count, `counts=0` returns 409 `KEY_USED`. 6.5 duplicate-key row re-runs the lookup. The page key lifecycle is in 6.5 (generate on load, delete after 200 or 409 `ALREADY_REDEEMED`/`KEY_USED`, keep after 5xx). AC6 restated "different key", AC11 extended, tests 23. |
| 2 | MEDIUM: media authorization SQL unspecified, `INSTR` match | Fixed. 6.2a has the full statement with whole-string equality on `p.image` and on `json_each` over `colors_json`, plus the stored colour shape. AC26, test 24. |
| 3 | MEDIUM: captcha rule stated three ways | Fixed. One rule: open portals always require captcha. 5.5 CHECK, 8.6 `captchaRequired` is not an input, 9.1 no toggle, `setStatus` blocks activation without Turnstile keys, `open`/`submit` return 503 without keys. AC24, AC25, test 25, decision 25. |
| 4 | NIT: `saveProduct` fix vs lock clause, bind numbering, NULL rows | Fixed in 8.5: `existing` loads `client_id`, 409 before the guard, `client_id` is `?26`, lock gains `AND (client_id IS ?26 OR client_id IS NULL)`, `DO UPDATE` only fills a NULL. Test 26. |
| 5 | NIT: `/api/showcase/media` not tenant-scoped | Accepted and recorded (8.5 row, decision 23), with the closing query given. |
| 6 | NIT: `rotateToken` leaves `email_status='sent'` | Fixed in 8.6: resets `email_status`, `email_sent_at`, `first_opened_at`, `sending_at`. Test 27. |
| 7 | NIT: unverified WAF free-tier claim | Fixed in 10.2: wording is conditional on the plan and says it was not verified. |
| 8 | NIT: `[redeem:...]` notes prefix; voided rows vs the 1000 cap | Fixed. 8.5 `splitNotes` gets a `[redeem:<portalId>]` branch; 8.6 states voided rows count. |

### Addendum round 4 (provisional inventory commits)

All six conflicts are answered in 8.7: client scoping of the list and the reset, moving the list logic into `_lib/inventory-admin.js`, the open-reservation guard with an atomic tripwire, the item pool from the catalog, the full list of inventory queries touched (no new inventory column), and `IVR_ORDER_RE` staying SHOWCASE-only. Decisions 24 and 26, AC27, test 28.

### Previous review (revision 3: 1 HIGH, 5 NIT)

| # | Finding | Response |
|---|---|---|
| 1 | HIGH: `admin/orders.js` local `isValidOrderNo` rejects `RDM-` | Fixed. 8.5 deletes the local regex and validator (lines 22-23) and uses the shared `isKnownOrderNo` at lines 136 and 342. The same function replaces the copies in `shipments.js` and `shipments/upload.js` and is used by the Showcase admin orders route. Renamed from `isShipmentOrderNo`. Unit test and AC20 added. |
| 2 | NIT: only `pending` can reach cancel | Fixed in 8.5: "`pending` or `processing`". Test 11 adds the `processing` variant. |
| 3 | NIT: `max_total_orders` pre-check can overshoot | Fixed by stating it as a soft cap (5.5, 6.4 step 5, UI text, test 13). A hard cap was considered and rejected: a conditional claim insert cannot roll back the order and reservation already in the batch. |
| 4 | NIT: AC14 vs one-segment route | Fixed. AC17 says "single path segment", and `/r/a/b` is expected to be a static 404 (test 18). |
| 5 | NIT: admin page count | Fixed. 8.5 says 7 files in total including `index.html`, and the new `redemption.html` is an 8th file. |
| 6 | NIT: `items_json` currency unspecified | Fixed. `currency: null` (5.6); `itemsCurrency()` returns null for it; the Orders page shows "No charge" for redemption totals. |

### Addendum (user)

| Addendum item | Where handled |
|---|---|
| Three tiers, Showcase admin in `showcase/`, own customer accounts, never in `admin_users`, never reaches `/api/admin/*` | 2, 2.1, 9.1, 11, AC2, AC3 |
| New `/api/showcase-admin/*`, server-side membership and client scope, reuse `_lib` logic | 2.1, 2.2, 8.1, 8.2 |
| Modules: Orders, Inventory, Redemption only | 8.3, 8.4, 8.6, decision 13 |
| `clients` table, `client_id` on all Showcase data, backfill to HRT | 5.2, 5.3 (the `clients` table exists from 0011); redemption tables in 5.5 |
| Per-client catalog and inventory; interaction with `inventory.sku` PK and `UNIQUE(product_id,color_id,size)` | 5.4, decision 12 |
| New migration, no edit of 0013 | 0014 and 0015, 5.1 |
| Cross-client denial tests | AC4, tests 2-5, scope lint test |
| Branding and client wizard out of scope | Section 1 out of scope |
| Portals belong to a client, screens in Showcase admin, Cyndi view says read-only | 5.5, 9, decision 14 |
| `RDM-` numbers valid everywhere including the local validator and Showcase admin orders | 8.2, 8.5, AC20 |
| Earlier decisions kept (system emails, caps, reserve/release, Type B approval) | 0, 6.4, 7, 8.6 |
| `admin_users` idea dropped; `scripts/make_admin_user.mjs` unused | A1, 11 |
| Migration plan, backfill, rollback note, manual D1 steps, risks | 15 |

### Earlier reviews

The first review's 11 findings (public images and CSP, shipment upload validator, email state machine, approver access, throttle ordering, cancel/delete of redemption orders, ASSETS redirect, `invResolveOrderLines` shape, email validation, item losing all variants, missing-table classification) stay fixed as in revision 2; their solutions are in 6.1-6.5, 7.4, 8.5 and 10.2. The approver-access fix changed: `rdmCan` is replaced by Showcase admin membership (section 11).

Unverified items carried as build steps, not assumptions: Resend API details (7.1), Turnstile `siteverify` and test keys, the exact D1 text for partial-unique-index violations, `env.ASSETS.fetch('/redeem')` on Pages (A5), and ARCC guidance.
