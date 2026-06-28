# Coffee POS — Database Design

A living document describing the database design for the coffee shop POS app, the
decisions behind it, and *why* each decision was made. Updated as the design
progresses.

- **Engine:** PostgreSQL
- **Scale:** Multi-location / franchise from day one
- **Connectivity:** Offline-first (terminals must work during outages and sync later)
- **v1 scope:** core ordering + payments, inventory/stock, discounts & promotions,
  cash drawer & shifts

> **How to read this doc:** each section first explains the *idea* and the *why*,
> then shows the DDL. The "Decision log" near the top is the quick reference for
> what we chose and the alternatives we rejected.

---

## Status / progress

| Step | Area | Status |
|-----:|------|--------|
| 1 | Conventions | ✅ Done |
| 2 | Organization & locations | ✅ Done |
| 3 | Catalog (products, modifiers, pricing) | ✅ Done |
| 4 | Orders & payments | ✅ Done |
| 5 | Discounts & promotions | ✅ Done |
| 6 | Inventory (movement ledger) | ✅ Done |
| 7 | Cash drawer & shifts | ⏳ Next |
| 8 | Auth & permissions | ☐ Planned |
| 9 | Cross-cutting (constraints, indexes, sync layer) | ☐ Planned |
| 10 | Migrations & seed data | ☐ Planned |

---

## Decision log

| # | Decision | Choice | Why / alternative rejected |
|---|----------|--------|----------------------------|
| D1 | Database engine | **PostgreSQL** | Best constraints, partial indexes, strong concurrency. |
| D2 | Deployment scale | **Multi-location / franchise** | Org→location hierarchy, location-scoped data from day one. |
| D3 | Connectivity | **Offline-first** | Terminals operate during outages; drives D4–D7. |
| D4 | Primary keys | **UUIDv7 everywhere** | Client-generatable (offline) + time-ordered (index locality). Rejected auto-increment (needs central coordinator) and UUIDv4 (random → poor index performance). |
| D5 | Money representation | **Integer minor units (cents) in BIGINT + currency code** | No float rounding; merges to-the-cent across offline nodes; multi-currency ready. Rejected DECIMAL (needs disciplined rounding) and floats (never). |
| D6 | Deletes | **Soft delete (`deleted_at` tombstone), never hard delete** | Hard deletes make offline conflicts unresolvable. |
| D7 | Catalog vs pricing scope | **Shared master catalog, per-location pricing & availability** | A "Latte" is defined once; price/availability live per location. |
| D8 | Inventory model | **Movement ledger (sum of movements), not a mutable counter** | Counters can't merge offline; ledgers do. (Built in Step 6.) |
| D9 | Modifier selection rules (min/max) | **On `product_modifiers` (per product)** | Same group can be required on one product, optional on another. Rejected putting rules on the global `modifiers` row. |
| D10 | Modifier-option price deltas | **Per-location (`location_modifier_options`)** | Currency-safe and consistent with per-location product pricing. Rejected a single global delta. |
| D11 | Order line data | **Snapshot + soft FK to catalog** | Lines copy name/price (self-describing forever) AND keep `product_id`/`modifier_option_id` for analytics. Display uses snapshot; reports join. Rejected pure snapshot (analytics need name-matching). |
| D12 | Order totals | **Stored frozen + DB CHECK** | Totals are snapshots; storing them prevents historical drift when prices/tax change. CHECK enforces `total = subtotal - discount + tax`. Rejected compute-on-read (unsafe for money). |
| D13 | Order status | **Status column + append-only `order_status_history`** | Column = fast current-state queries; history = audit trail (voids/refunds) and offline reconciliation aid. Rejected status-column-only (no audit). |
| D14 | Human order numbers | **Terminal-local ticket code** (e.g. `2-042`) | Generated instantly offline, resets daily, collision-free by construction. Rejected server-assigned sequence (NULL until sync, no offline receipt number). |
| D15 | Discount scope | **Order + line, one table** (`order_discounts.order_item_id` nullable) | NULL = whole-order, set = single line. One clean table covers both. Rejected order-level-only (can't comp an item). |
| D16 | Discount types | **percentage + fixed_amount + set_price** | Covers % off, € off, and "this item is now €X". Applications always store a *resolved deduction* so money math stays uniform. |
| D17 | Predefined vs ad-hoc | **Both** (`discount_id` nullable, `reason` required when ad-hoc) | Catalog promos plus one-off manager discounts. CHECK forces a reason on ad-hoc (anti-shrinkage). Authorization in Step 8. |
| D18 | Promo machinery (v1) | **Moderate** | code, validity window, min spend, is_active, allow_stacking. Rejected full (usage/per-customer limits need loyalty accounts, not in v1). |
| D19 | Inventory tracking model | **Ingredients + recipes (BOM)** | Track raw materials; products consume them via `product_ingredients`. Retail goods = 1-row recipe. Rejected finished-goods-only (too coarse for a café). |
| D20 | Stock representation | **Append-only ledger (`stock_movements`); stock = SUM of deltas** | Mutable counters can't merge offline; an append-only ledger does (commutative). Quantities are integers in a base unit (g/ml/each), mirroring cents. |
| D21 | Movement types | **Full set** | sale, restock, waste, transfer_in/out, count_adjustment, return. DB CHECK enforces the sign per type. Rejected minimal (no waste/transfers). |
| D22 | Stock-on-hand storage | **Derived cache `stock_balances` + trigger** | Projection of the ledger → O(1) reads; rebuildable from movements (ledger = source of truth). Rejected compute-on-read (slows as ledger grows). |
| D23 | Costing | **Capture cost on restocks + moving average** | Store `unit_cost_cents` on restocks (offline-safe raw data); compute moving-average COGS server-side in reporting. Rejected FIFO/lot (overkill) and no-costing (no margins). |

---

## Step 1 — Conventions

Applied to **every** table unless noted.

### Primary keys — UUIDv7
```sql
id UUID PRIMARY KEY DEFAULT uuid_generate_v7()
```
- **Client-generatable:** an offline terminal mints the final ID itself — no
  "temporary ID then renumber" on sync.
- **Time-ordered (v7):** the high bits encode a timestamp, so new rows append near
  the "end" of B-tree indexes instead of scattering random writes (which UUIDv4
  does). You get coordination-free uniqueness *and* near auto-increment insert speed.
- *Implementation note:* Postgres 18 ships native `uuidv7()`. Until then use an
  extension (e.g. `pg_uuidv7`) or generate v7 in the app. Pinned in Step 10.

### Money — integer minor units
```sql
price_cents BIGINT  NOT NULL,
currency    CHAR(3) NOT NULL DEFAULT 'USD'   -- ISO 4217
```
- `$4.50` → `450`. Integer arithmetic = zero rounding drift, which matters when
  summing many lines across offline terminals that must reconcile to the cent.
- `BIGINT` to never overflow (daily takings, reports).
- `currency` carried now (cheap) to avoid a brutal cross-border migration later.
  Rule: **one currency per location**, so no conversion logic in v1.

### The sync envelope — on every table
```sql
created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
deleted_at TIMESTAMPTZ            -- NULL = alive; a timestamp = tombstoned
```
- **`TIMESTAMPTZ`, never `TIMESTAMP`** — store absolute instants (UTC). A naive
  timestamp is ambiguous across time zones, which a franchise will have.
- **`deleted_at` = soft delete.** Live-data queries filter `WHERE deleted_at IS NULL`.
- **`updated_at`** is how the sync layer detects changed rows; kept honest by a
  trigger (added in Step 9), not by trusting app code.

### Location scoping
- Business/transactional tables carry `location_id UUID NOT NULL REFERENCES locations(id)`.
- Shared catalog tables (master `products`, `modifiers`) carry `organization_id`
  instead; pricing/availability live in per-location link tables.

### Naming
- `snake_case`; tables **plural**, columns singular.
- Join tables: alphabetical pair (`product_modifiers`).
- FKs: `<singular>_id` (`location_id`, `created_by_user_id`).
- Money columns end `_cents`; timestamps end `_at`; booleans read as a fact (`is_void`).

### Recurring pattern — soft-delete-aware uniqueness & FK indexes
Plain `UNIQUE`/indexes conflict with tombstones. We use **partial** indexes:
```sql
CREATE UNIQUE INDEX uq_x ON t(parent_id, lower(name)) WHERE deleted_at IS NULL;
CREATE INDEX        idx_x ON t(parent_id)             WHERE deleted_at IS NULL;
```
Only live rows are indexed → smaller/faster, and reusing a tombstoned name is allowed.

---

## Step 2 — Organization & locations

The tenancy spine everything hangs off:

```
organization  (the franchise / tenant — the security boundary)
   └── location  (a physical shop: owns currency, timezone, address)
          └── terminal  (a POS device / register)
```

- **`organizations`** is the tenant boundary — the line franchise data must never
  cross. Everything scoped to a location is transitively scoped to an org.
- **`locations`** own what varies per shop: `currency`, **`timezone`** (IANA, e.g.
  `Europe/Berlin`). Store instants in UTC, but compute "today's sales" against the
  shop's local day boundary — so the zone is needed.
- **`terminals`** exist because offline sync must know *which device* a change came
  from (conflict attribution) and because a cash drawer belongs to a register
  (Step 7). Would be skippable without offline/cash needs — but we have both.

```sql
CREATE TABLE organizations (
    id         UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    name       TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at TIMESTAMPTZ
);

CREATE TABLE locations (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    timezone        TEXT NOT NULL DEFAULT 'UTC',    -- IANA zone
    currency        CHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE TABLE terminals (
    id          UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    location_id UUID NOT NULL REFERENCES locations(id),
    name        TEXT NOT NULL,            -- 'Front counter 1'
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at  TIMESTAMPTZ
);

CREATE INDEX idx_locations_org      ON locations(organization_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_terminals_location ON terminals(location_id)     WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX uq_locations_org_name
    ON locations(organization_id, lower(name)) WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX uq_terminals_location_name
    ON terminals(location_id, lower(name))     WHERE deleted_at IS NULL;
```

*Deferred:* address/contact fields on `locations` (pure CRUD, non-breaking to add).

---

## Step 3 — Catalog

Principle: **master catalog = identity (what a thing *is*); location layer = money +
availability (what it costs / whether it's sold here).**

```
organization
  ├── categories
  ├── products ──< product_modifiers >── modifiers ──< modifier_options
  │      │                                                   │
  │      └─ location_products              location_modifier_options
  │           (price_cents, is_available)    (price_delta_cents, is_available)
  │           per location                    per location
```

- **`products`** define an item once for the whole org and have **no price**.
- **`location_products`** hold the per-location `price_cents` + `is_available`
  (also the *sold-out* toggle). No row = not offered at that location.
- **`modifiers`** are reusable groups ("Milk", "Size"); **`modifier_options`** are
  the choices ("Oat", "Large") and carry **identity only, no price** (D10).
- **`location_modifier_options`** hold the per-location `price_delta_cents` +
  availability — mirrors `location_products`, keeping one consistent principle.
- **`product_modifiers`** link products to modifier groups **and** hold the
  selection rules `min_select`/`max_select` **per product** (D9). E.g. Size =
  required single choice (`min 1, max 1`); Syrup = optional up to 3 (`min 0, max 3`).

> **Catalog is the *current* menu, never the historical record.** Past orders must
> not change when the menu does — Step 4 snapshots names/prices onto order lines.

```sql
CREATE TABLE categories (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    sort_order      INT  NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE TABLE products (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    category_id     UUID NOT NULL REFERENCES categories(id),
    name            TEXT NOT NULL,
    sort_order      INT  NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE TABLE location_products (
    id           UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    location_id  UUID NOT NULL REFERENCES locations(id),
    product_id   UUID NOT NULL REFERENCES products(id),
    price_cents  BIGINT NOT NULL CHECK (price_cents >= 0),
    is_available BOOLEAN NOT NULL DEFAULT true,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at   TIMESTAMPTZ
);

CREATE TABLE modifiers (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,         -- "Milk", "Size"
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE TABLE modifier_options (
    id          UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    modifier_id UUID NOT NULL REFERENCES modifiers(id),
    name        TEXT NOT NULL,             -- "Oat", "Large" (no price here)
    sort_order  INT  NOT NULL DEFAULT 0,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at  TIMESTAMPTZ
);

CREATE TABLE location_modifier_options (
    id                 UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    location_id        UUID NOT NULL REFERENCES locations(id),
    modifier_option_id UUID NOT NULL REFERENCES modifier_options(id),
    price_delta_cents  BIGINT NOT NULL DEFAULT 0,   -- may be negative
    is_available       BOOLEAN NOT NULL DEFAULT true,
    created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at         TIMESTAMPTZ
);

CREATE TABLE product_modifiers (
    id          UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    product_id  UUID NOT NULL REFERENCES products(id),
    modifier_id UUID NOT NULL REFERENCES modifiers(id),
    min_select  INT NOT NULL DEFAULT 0,
    max_select  INT NOT NULL DEFAULT 1,
    sort_order  INT NOT NULL DEFAULT 0,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at  TIMESTAMPTZ,
    CHECK (min_select >= 0),
    CHECK (max_select >= min_select),
    CHECK (max_select >= 1)
);

-- Indexes & uniqueness
CREATE INDEX idx_products_category       ON products(category_id)                   WHERE deleted_at IS NULL;
CREATE INDEX idx_products_org            ON products(organization_id)               WHERE deleted_at IS NULL;
CREATE INDEX idx_modifier_options_mod    ON modifier_options(modifier_id)           WHERE deleted_at IS NULL;
CREATE INDEX idx_product_modifiers_prod  ON product_modifiers(product_id)           WHERE deleted_at IS NULL;
CREATE INDEX idx_location_products_loc   ON location_products(location_id)          WHERE deleted_at IS NULL;
CREATE INDEX idx_location_mod_options_loc ON location_modifier_options(location_id) WHERE deleted_at IS NULL;

CREATE UNIQUE INDEX uq_location_products
    ON location_products(location_id, product_id) WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX uq_location_mod_options
    ON location_modifier_options(location_id, modifier_option_id) WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX uq_product_modifiers
    ON product_modifiers(product_id, modifier_id) WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX uq_categories_name       ON categories(organization_id, lower(name)) WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX uq_products_name         ON products(organization_id, lower(name))   WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX uq_modifiers_name        ON modifiers(organization_id, lower(name))  WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX uq_modifier_options_name ON modifier_options(modifier_id, lower(name)) WHERE deleted_at IS NULL;
```

---

## Step 4 — Orders & payments

Guiding principle: **an order is a permanent financial record, not a view of current
data.** Once placed it must be fully self-describing and immutable in its money, even
if the menu, prices, tax rates, or staff change later. The catalog is "now"; the
order is "what happened then."

```
orders            (header: who/where/when, status, frozen totals, ticket code)
  ├── order_items            (line: product snapshot + soft FK, qty, line total)
  │     └── order_item_modifiers   (chosen options: snapshots + delta)
  └── payments               (one or more: split tender, refunds)
orders ──< order_status_history   (append-only audit log)
```

Key points:
- **Snapshots (D11):** lines copy `product_name`, `unit_price_cents`; modifiers copy
  `modifier_name`, `option_name`, `price_delta_cents`. A receipt reprinted years later
  is identical even if the catalog changed. Soft FKs (`product_id`,
  `modifier_option_id`) are kept for analytics only — never for display.
- **Frozen totals (D12):** money stored on the order, computed on the terminal,
  verified on sync. A row-level CHECK enforces `total = subtotal - discount + tax`.
  (Cross-row "subtotal = sum of lines" → trigger in Step 9 + sync verification.)
- **`line_total_cents`** = `(unit_price_cents + sum of chosen modifier deltas) * quantity`.
- **Status (D13):** `status` column for current state + append-only
  `order_status_history` for the audit trail (who voided/refunded, when, which terminal).
- **Ticket code (D14):** terminal-local, e.g. `2-042`, generated offline, unique per
  `(location, business_date)`, resets daily.
- **`business_date`:** the location-local calendar day (derived from `created_at` +
  location timezone) so daily counters and reports are correct across time zones.
- **Payments:** support **split tender** (many rows per order) and **refunds** (via
  `type`); `amount_cents` is always positive and `type` gives direction (cleaner CHECKs,
  no accidental negative payments). Cash captures `tendered`/`change`.

### Mutable state vs immutable events
`order_status_history` is **append-only** — rows are never edited or deleted — so it
deliberately omits `updated_at`/`deleted_at`. Distinguishing mutable-state tables from
immutable-event logs is a recurring design tool (reused for the inventory ledger, Step 6).

```sql
CREATE TABLE orders (
    id                 UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    location_id        UUID NOT NULL REFERENCES locations(id),
    terminal_id        UUID NOT NULL REFERENCES terminals(id),
    created_by_user_id UUID NOT NULL REFERENCES users(id),   -- users built in Step 8
    currency           CHAR(3) NOT NULL,           -- snapshot of location currency at sale
    ticket_code        TEXT NOT NULL,              -- human-facing, generated offline (e.g. '2-042')
    business_date      DATE NOT NULL,              -- location-local calendar day
    status             TEXT NOT NULL DEFAULT 'open',
    subtotal_cents     BIGINT NOT NULL DEFAULT 0 CHECK (subtotal_cents >= 0),
    discount_cents     BIGINT NOT NULL DEFAULT 0 CHECK (discount_cents >= 0),
    tax_cents          BIGINT NOT NULL DEFAULT 0 CHECK (tax_cents      >= 0),
    total_cents        BIGINT NOT NULL DEFAULT 0 CHECK (total_cents    >= 0),
    note               TEXT,
    created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at         TIMESTAMPTZ,
    CONSTRAINT chk_orders_status CHECK (status IN
        ('open','held','completed','voided','refunded','partially_refunded')),
    CONSTRAINT chk_orders_total  CHECK (total_cents = subtotal_cents - discount_cents + tax_cents)
);

CREATE TABLE order_items (
    id               UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    order_id         UUID NOT NULL REFERENCES orders(id),
    product_id       UUID NOT NULL REFERENCES products(id),  -- soft FK: reporting only
    product_name     TEXT   NOT NULL,                         -- snapshot
    unit_price_cents BIGINT NOT NULL CHECK (unit_price_cents >= 0),
    quantity         INT    NOT NULL CHECK (quantity > 0),
    line_total_cents BIGINT NOT NULL CHECK (line_total_cents >= 0),
    note             TEXT,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at       TIMESTAMPTZ
);

CREATE TABLE order_item_modifiers (
    id                 UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    order_item_id      UUID NOT NULL REFERENCES order_items(id),
    modifier_option_id UUID NOT NULL REFERENCES modifier_options(id),  -- soft FK
    modifier_name      TEXT   NOT NULL,        -- snapshot, "Milk"
    option_name        TEXT   NOT NULL,        -- snapshot, "Oat"
    price_delta_cents  BIGINT NOT NULL,        -- snapshot (may be negative)
    created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at         TIMESTAMPTZ
);

CREATE TABLE payments (
    id                 UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    order_id           UUID NOT NULL REFERENCES orders(id),
    location_id        UUID NOT NULL REFERENCES locations(id),  -- denormalized for reporting
    type               TEXT NOT NULL DEFAULT 'payment',
    method             TEXT NOT NULL,
    amount_cents       BIGINT NOT NULL CHECK (amount_cents > 0), -- positive; type gives direction
    currency           CHAR(3) NOT NULL,
    tendered_cents     BIGINT,   -- cash only
    change_cents       BIGINT,   -- cash only
    created_by_user_id UUID NOT NULL REFERENCES users(id),
    created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at         TIMESTAMPTZ,
    CONSTRAINT chk_payments_type   CHECK (type   IN ('payment','refund')),
    CONSTRAINT chk_payments_method CHECK (method IN ('cash','card','mobile','voucher','other')),
    CONSTRAINT chk_payments_tender CHECK (tendered_cents IS NULL OR tendered_cents >= amount_cents),
    CONSTRAINT chk_payments_change CHECK (
        change_cents IS NULL OR tendered_cents IS NULL OR change_cents = tendered_cents - amount_cents)
);

CREATE TABLE order_status_history (   -- append-only: no updated_at / deleted_at
    id                 UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    order_id           UUID NOT NULL REFERENCES orders(id),
    from_status        TEXT,                       -- NULL for initial 'open'
    to_status          TEXT NOT NULL,
    reason             TEXT,
    changed_by_user_id UUID NOT NULL REFERENCES users(id),
    terminal_id        UUID NOT NULL REFERENCES terminals(id),
    created_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Indexes
CREATE INDEX idx_orders_loc_date  ON orders(location_id, business_date) WHERE deleted_at IS NULL;
CREATE INDEX idx_orders_status    ON orders(location_id, status)        WHERE deleted_at IS NULL;
CREATE INDEX idx_orders_terminal  ON orders(terminal_id)               WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX uq_orders_ticket
    ON orders(location_id, business_date, ticket_code) WHERE deleted_at IS NULL;
CREATE INDEX idx_order_items_order       ON order_items(order_id)               WHERE deleted_at IS NULL;
CREATE INDEX idx_order_items_product     ON order_items(product_id)             WHERE deleted_at IS NULL;
CREATE INDEX idx_oi_modifiers_item       ON order_item_modifiers(order_item_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_payments_order          ON payments(order_id)                  WHERE deleted_at IS NULL;
CREATE INDEX idx_payments_loc            ON payments(location_id)               WHERE deleted_at IS NULL;
CREATE INDEX idx_order_status_hist_order ON order_status_history(order_id);
```

---

## Step 5 — Discounts & promotions

Reuses the Step 4 pattern: a **definition** layer (editable promo catalog) and an
**application** layer (snapshotted onto the order, immutable).

```
discounts        the promo catalog ("Happy Hour 10%", "Staff 20%", "€2 pastry")
order_discounts  what was actually applied to an order/line (snapshot)
```

Key points:
- **Discounts are separate and additive** — they do NOT alter `line_total_cents`
  (which stays pre-discount). Instead: `order.discount_cents = sum(order_discounts.amount_cents)`,
  and the Step 4 CHECK `total = subtotal - discount + tax` still holds.
- **Applications store a *resolved deduction*** (D16): whatever the type, the cents
  actually taken off are snapshotted in `order_discounts.amount_cents`. A `set_price`
  "€2 item" is stored as the equivalent "−€X" deduction, so all rollup math is uniform.
  `type`/`rate_bps` are snapshotted only for receipt wording.
- **Scope (D15):** `order_discounts.order_item_id` NULL = whole order, set = one line.
- **Predefined + ad-hoc (D17):** `discount_id` nullable; CHECK requires a `reason` when
  there's no catalog row (no silent unexplained discounts).
- **`chk_discounts_value`** keeps each type paired with exactly its value column
  (percentage↔rate_bps, fixed/set-price↔amount_cents) — the DB rejects malformed promos.
- **Eligibility lives in app/sync, not schema:** `allow_stacking`, validity window,
  and `min_subtotal_cents` are checked when applying; the DB stores the resulting snapshot.

```sql
CREATE TABLE discounts (
    id                 UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    organization_id    UUID NOT NULL REFERENCES organizations(id),
    name               TEXT NOT NULL,
    type               TEXT NOT NULL,                 -- 'percentage' | 'fixed_amount' | 'set_price'
    rate_bps           BIGINT,   -- percentage only: basis points, 1000 = 10.00%
    amount_cents       BIGINT,   -- fixed_amount: deduction; set_price: target price
    code               TEXT,
    starts_at          TIMESTAMPTZ,
    ends_at            TIMESTAMPTZ,
    min_subtotal_cents BIGINT CHECK (min_subtotal_cents >= 0),
    allow_stacking     BOOLEAN NOT NULL DEFAULT true,
    is_active          BOOLEAN NOT NULL DEFAULT true,
    created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at         TIMESTAMPTZ,
    CONSTRAINT chk_discounts_type CHECK (type IN ('percentage','fixed_amount','set_price')),
    CONSTRAINT chk_discounts_value CHECK (
        (type = 'percentage' AND rate_bps IS NOT NULL AND rate_bps BETWEEN 0 AND 10000
                             AND amount_cents IS NULL)
     OR (type IN ('fixed_amount','set_price') AND amount_cents IS NOT NULL AND amount_cents >= 0
                             AND rate_bps IS NULL)
    ),
    CONSTRAINT chk_discounts_window CHECK (ends_at IS NULL OR starts_at IS NULL OR ends_at >= starts_at)
);

CREATE TABLE order_discounts (
    id                 UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    order_id           UUID NOT NULL REFERENCES orders(id),
    order_item_id      UUID REFERENCES order_items(id),     -- NULL = whole-order discount
    discount_id        UUID REFERENCES discounts(id),       -- soft FK; NULL = ad-hoc
    name               TEXT   NOT NULL,                      -- snapshot
    type               TEXT   NOT NULL,                      -- snapshot
    rate_bps           BIGINT,                               -- snapshot (percentage, for wording)
    amount_cents       BIGINT NOT NULL CHECK (amount_cents > 0),  -- resolved deduction applied
    reason             TEXT,                                 -- required for ad-hoc
    applied_by_user_id UUID NOT NULL REFERENCES users(id),
    created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at         TIMESTAMPTZ,
    CONSTRAINT chk_order_discounts_type  CHECK (type IN ('percentage','fixed_amount','set_price')),
    CONSTRAINT chk_order_discounts_adhoc CHECK (discount_id IS NOT NULL OR reason IS NOT NULL)
);

CREATE INDEX idx_discounts_org ON discounts(organization_id) WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX uq_discounts_code
    ON discounts(organization_id, lower(code))
    WHERE code IS NOT NULL AND deleted_at IS NULL;
CREATE INDEX idx_order_discounts_order    ON order_discounts(order_id)      WHERE deleted_at IS NULL;
CREATE INDEX idx_order_discounts_item     ON order_discounts(order_item_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_order_discounts_discount ON order_discounts(discount_id)   WHERE deleted_at IS NULL;
```

---

## Step 6 — Inventory (movement ledger)

**Core idea:** stock is a *derived sum*, never a stored counter. We keep an
append-only **ledger** of movements; current stock = `SUM(quantity_delta)`. This is
what survives offline — two terminals each append "−1"; on sync both rows live and the
math is simply correct. Addition is commutative, so sync order never matters. A mutable
counter has no events to replay and cannot do this.

### Three table archetypes (design vocabulary)
| Archetype | Examples | Column signature |
|---|---|---|
| **Entity** (mutable state) | products, orders, discounts, inventory_items, product_ingredients | UUID PK + full envelope |
| **Event log** (immutable) | order_status_history, stock_movements | UUID PK + `created_at` only |
| **Projection** (rebuildable cache) | stock_balances | natural key + `updated_at` |

Knowing a table's archetype tells you exactly which columns it needs.

Key points:
- **Quantities are integers in a base unit** (g, ml, each), same reasoning as cents —
  no float drift, clean offline summation. kg→g etc. converted at data entry.
- **Recipes/BOM** (`product_ingredients`): a sale appends one `sale` movement per
  ingredient (`−quantity × order_qty`), stamped with `order_id`. Retail goods = 1-row recipe.
- **Sign integrity:** `chk_movement_sign` makes the DB reject wrong-direction data
  (restock must be +, sale must be −, etc.).
- **Transfers** between locations = two rows sharing `transfer_group_id` (out − / in +).
- **Corrections never edit history** — append a compensating movement (return /
  count_adjustment), preserving the audit trail.
- **`stock_balances`** is a disposable projection (no UUID, no soft-delete); maintained
  by a trigger (Step 9) and rebuildable from the ledger at any time.
- **Costing:** `unit_cost_cents` captured on restocks (offline-safe); moving-average
  COGS computed server-side in reporting.

```sql
CREATE TABLE inventory_items (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,          -- "Espresso beans", "Whole milk", "12oz cup"
    unit            TEXT NOT NULL,          -- base unit: 'g', 'ml', 'each'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE TABLE stock_movements (   -- append-only event log
    id                 UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    location_id        UUID   NOT NULL REFERENCES locations(id),
    inventory_item_id  UUID   NOT NULL REFERENCES inventory_items(id),
    quantity_delta     BIGINT NOT NULL CHECK (quantity_delta <> 0),  -- base units; sign per type
    type               TEXT   NOT NULL,
    reason             TEXT,
    unit_cost_cents    BIGINT CHECK (unit_cost_cents >= 0),  -- set on restock
    order_id           UUID REFERENCES orders(id),           -- links a 'sale' to its order
    transfer_group_id  UUID,                                 -- pairs transfer_out + transfer_in
    created_by_user_id UUID REFERENCES users(id),
    terminal_id        UUID REFERENCES terminals(id),
    created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_movement_type CHECK (type IN
        ('sale','restock','waste','transfer_in','transfer_out','count_adjustment','return')),
    CONSTRAINT chk_movement_sign CHECK (
        (type = 'restock'      AND quantity_delta > 0) OR
        (type = 'sale'         AND quantity_delta < 0) OR
        (type = 'waste'        AND quantity_delta < 0) OR
        (type = 'transfer_in'  AND quantity_delta > 0) OR
        (type = 'transfer_out' AND quantity_delta < 0) OR
        (type = 'return'       AND quantity_delta > 0) OR
        (type = 'count_adjustment')
    )
);

CREATE TABLE stock_balances (    -- projection / rebuildable cache (natural key, no envelope)
    location_id       UUID   NOT NULL REFERENCES locations(id),
    inventory_item_id UUID   NOT NULL REFERENCES inventory_items(id),
    quantity_on_hand  BIGINT NOT NULL DEFAULT 0,    -- = SUM(stock_movements.quantity_delta)
    reorder_threshold BIGINT,                       -- per-location low-stock alert level
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (location_id, inventory_item_id)
);

CREATE TABLE product_ingredients (   -- recipe / BOM
    id                UUID PRIMARY KEY DEFAULT uuid_generate_v7(),
    product_id        UUID   NOT NULL REFERENCES products(id),
    inventory_item_id UUID   NOT NULL REFERENCES inventory_items(id),
    quantity          BIGINT NOT NULL CHECK (quantity > 0),  -- base units per 1 unit sold
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at        TIMESTAMPTZ
);

CREATE UNIQUE INDEX uq_inventory_items_name
    ON inventory_items(organization_id, lower(name)) WHERE deleted_at IS NULL;
CREATE INDEX idx_stock_movements_item_loc ON stock_movements(location_id, inventory_item_id);
CREATE INDEX idx_stock_movements_order    ON stock_movements(order_id)          WHERE order_id IS NOT NULL;
CREATE INDEX idx_stock_movements_transfer ON stock_movements(transfer_group_id) WHERE transfer_group_id IS NOT NULL;
CREATE INDEX idx_stock_balances_location  ON stock_balances(location_id);
CREATE UNIQUE INDEX uq_product_ingredients
    ON product_ingredients(product_id, inventory_item_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_product_ingredients_product ON product_ingredients(product_id) WHERE deleted_at IS NULL;
```

**Stock-balance maintenance trigger (formalized in Step 9):**
```sql
INSERT INTO stock_balances (location_id, inventory_item_id, quantity_on_hand)
VALUES (NEW.location_id, NEW.inventory_item_id, NEW.quantity_delta)
ON CONFLICT (location_id, inventory_item_id)
DO UPDATE SET quantity_on_hand = stock_balances.quantity_on_hand + NEW.quantity_delta,
              updated_at = now();
```

---

## Open items / to revisit
- Step 9: a shared trigger to auto-maintain `updated_at` on every table.
- Step 9: the sync/change-log layer and conflict-resolution policy.
- Step 10: how UUIDv7 is generated (extension vs app vs PG18 native).
- Category nesting (self-referencing `parent_id`) — only if needed later.
- `locations` address/contact block — add when convenient.
- **Migration ordering:** `users` (Step 8) must be created *before* `orders`,
  `payments`, and `order_status_history`, which reference `users(id)`.
- Step 9: trigger to enforce `orders.subtotal_cents = sum(order_items.line_total_cents)`.
- Step 9: trigger to enforce `orders.discount_cents = sum(order_discounts.amount_cents)`.
- App/sync: discount eligibility (stacking, validity window, min spend) — not in schema.
- Deferred: per-location promo *targeting* (`location_discounts` link), like the catalog pattern.
- Step 9: trigger to maintain `stock_balances` from `stock_movements` (upsert shown above).
- App/sync: generating `sale` movements from order + recipe; voids/refunds append compensating movements.
- Deferred: per-location recipe overrides; purchase-unit↔stock-unit conversions (e.g. case→each).

---

*Last updated: through Step 6 (Inventory — movement ledger).*
