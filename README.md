# Coffee App

A multi-location, offline-first POS system for coffee shops.

- **Engine:** PostgreSQL
- **Scale:** Multi-location / franchise from day one
- **Connectivity:** Offline-first — terminals operate during outages and sync later

## Design highlights

The schema is built around a few principles that make it correct under offline,
multi-terminal operation:

- **UUIDv7 primary keys** — client-generatable so an offline terminal mints final IDs
  itself (no renumbering on sync), and time-ordered for good B-tree index locality.
- **Money as integer minor units** — amounts stored as cents in `BIGINT` with a currency
  code; zero floating-point rounding drift when summing across terminals that must
  reconcile to the cent.
- **Soft deletes, never hard deletes** — a `deleted_at` tombstone on every table so
  offline conflicts stay resolvable; live queries filter `WHERE deleted_at IS NULL`.
- **Snapshot + soft FK on financial records** — orders copy product names and prices at
  sale time, so a reprinted receipt is identical forever even if the menu changes, while
  keeping soft foreign keys for analytics.
- **Frozen totals with DB CHECK constraints** — order totals are stored, not recomputed,
  and a constraint enforces `total = subtotal − discount + tax`.
- **Inventory as an append-only ledger** — stock on hand is the `SUM` of immutable
  movement rows, not a mutable counter, so concurrent offline changes merge correctly
  (addition is commutative); a derived cache gives O(1) reads.
- **Shared master catalog, per-location pricing** — a product is defined once for the
  org; price and availability live in per-location link tables.

## Documentation

- [Database Design](docs/DATABASE_DESIGN.md) — living document covering the schema,
  the decisions behind it, and the reasoning for each (Steps 1–6 complete).
