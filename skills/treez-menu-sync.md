---
name: treez-menu-sync
description: Build and incrementally refresh a sellable dispensary menu from Treez - products, live stock, discounts and lab results - without missing inventory movement.
api: openapi/treez-dispensary-openapi.json
generated: '2026-08-30'
method: generated
source: https://code.treez.io/recipes/getting-products-available-for-sale
operations:
  - dispensary-product-get-all
  - dispensary-product-get-by-updated-date
  - dispensary-inventory-get-stock
  - dispensary-inventory-get
  - dispensary-discounts-get-all
  - get_dispensary-inventory-labs-package-id
  - location-get-all
---

# Sync a sellable menu out of Treez

Read-only. This is the flow behind every online menu, marketplace listing and analytics feed.
Base: `https://api-prod.treez.io/dispensary/v3/{dispensary}`.

## The trap to know before you start

`dispensary-product-get-by-updated-date` is the obvious delta feed, and Treez documents plainly
that a product's `last_updated_at` **only moves when product detail changes — inventory quantity
changes do not update it.** A menu built on the product delta alone will happily show sold-out
product for as long as nobody edits its description. Stock is a separate poll.

## Steps

1. **Seed the catalog.** `dispensary-product-get-all`
   (`GET /{dispensary}/product/product_list`). Useful filters, all query parameters:
   - `active` — `TRUE` (default), `FALSE` (deactivated only), or `ALL`
   - `category_type` — filter by product type
   - `ID` — up to **50** product ids per call; more returns `400 RESPONSE_LIMIT_EXCEEDS`
   - `above_threshold` — only products above the store's configured minimum visible inventory
   - `sellable_quantity_in_type` — `MEDICAL`, `ADULT` or `ALL`
   - `sellable_quantity_in_location` — restrict to one location
   - `include_discounts=FALSE` — drops discount detail; materially smaller and faster payloads
   Results are sorted by `name` asc, `product_id` asc.

2. **Poll product detail deltas.** `dispensary-product-get-by-updated-date`
   (`GET /{dispensary}/product/product_list/lastUpdated/after/{last_updated_at}`).
   Format the timestamp as `yyyy-MM-ddTHH:mm:ss.SSSXXX` (e.g. `2026-07-14T10:55:58.000-07:00`) —
   anything else returns `400 INVALID_DATE_FORMAT`.

3. **Poll stock separately, and more often.** `dispensary-inventory-get-stock`
   (`GET /{dispensary}/stock/getStock`) for sellable quantity, and `dispensary-inventory-get`
   (`GET /{dispensary}/packages`) when you need package-level detail. This is the step that keeps
   the menu honest.

4. **Pull discounts** with `dispensary-discounts-get-all`
   (`GET /{dispensary}/discount/all`) if you dropped them from the product payload in step 1.

5. **Attach lab results** with `get_dispensary-inventory-labs-package-id`
   (`GET /{dispensary}/inventory/labs/package_id`), keyed on the package. In most US markets a
   published potency/COA figure on a menu has to trace to the package actually on the shelf, so
   bind results to `package_id` / `package_label`, never to the product master.

6. **Map locations** with `location-get-all` (`GET /{dispensary}/location/list`) if you surface
   in-store availability by area.

## Conventions

- **Pagination is in the path**, not the query string: `.../page/{page}/pagesize/{pagesize}`.
  There is no `next` link, no total count and no cursor — you know you have reached the end when
  a page comes back short. Too large a `pagesize` returns `400 RESPONSE_LIMIT_EXCEEDS`.
- **No rate limits are published** — no `429`, no `RateLimit-*` headers, no documented ceiling.
  Pace polling conservatively and back off on any non-200.
- `product_id` is per-dispensary. The same physical product in two stores of the same organization
  is two ids unless you join through the central catalog service
  (`openapi/treez-catalog-openapi.json`).
