---
name: treez-catalog-product-onboarding
description: Create a product in the Treez central catalog with its SKUs, brand, subcategory, images and per-store price - the organization-level path, not the per-dispensary one.
api: openapi/treez-catalog-openapi.json
generated: '2026-08-30'
method: generated
source: https://code.treez.io/reference/catalog
operations:
  - catalog-stores-read
  - catalog-brands-read
  - catalog-categories-read
  - catalog-categories-read-1-1
  - catalog-product-search
  - catalog-product-create
  - catalog-sku-create
  - catalog-image-create
  - catalog-image-link
  - catalog-store-price-create
---

# Onboard a product into the Treez catalog

The catalog is an **organization-level service**, not a per-store one. Base:
`https://api-prod.treez.io/service/catalog`. Scope comes from the `oid` claim in your JWT, so the
same call lands in a different organization purely by re-signing the token.

Creating a product master does **not** create inventory. Stock arrives through receiving
(`treez-inbound-invoice-receiving`).

## Steps

1. **Read the stores you can write to.** `catalog-stores-read`
   (`GET /v3/organization-entity`) returns every entity (store) under the organization that signed
   the JWT. No query parameters needed — the token is the filter. Keep the `entityId` values; you
   need them for pricing.

2. **Resolve the taxonomy before you write.**
   - `catalog-brands-read` (`GET /v3/brand`)
   - `catalog-categories-read` (`GET /{ver}/product-category`)
   - `catalog-categories-read-1-1` (`GET /{ver}/resolved-subcategory`) — **use this one.** It
     returns the organization's *effective* subcategory set: every custom subcategory unioned with
     every global one, each row tagged `type: custom | canonical` with `isExcluded`, `isShadowed`
     and `shownInProductDropdown`. With no parameters it returns only rows where
     `shownInProductDropdown` is true — exactly the set a product picker should offer.

   When you assign the subcategory in step 4, use `id + type` from that response:
   `customSubCategoryId` for `custom` rows, `productSubCategoryId` for `canonical` rows. Getting
   this backwards is the most common validation failure on catalog writes.

3. **Check for an existing product first.** `catalog-product-search` (`POST /v3/search`). Catalog
   writes are not idempotent; a duplicate product master is materially harder to unwind than a
   duplicate lookup is to avoid — products have **no delete operation**.

4. **Create the product master.** `catalog-product-create` (`POST /product`).

5. **Create each SKU / variant.** `catalog-sku-create` (`POST /v3/sku`), referencing the
   `productId` from step 4.

6. **Attach images — a three-call sequence.**
   1. `catalog-image-create` (`POST /v3/product/image/create`) returns a **pre-signed, one-time**
      upload URL.
   2. `PUT` the image bytes to that URL (not a Treez API call; it is object storage).
   3. `catalog-image-link` (`POST /v3/imageDetail`) links the uploaded file to the variant/SKU.
   The pre-signed URL is single-use — if step 2 fails, go back to step 1 rather than retrying the
   same URL.

7. **Price it per store.** `catalog-store-price-create` (`POST /v3/entity-price`), one call per
   `entityId`. `catalog-store-price-update` (PATCH) and `catalog-store-price-delete` (DELETE)
   exist, so pricing is the one part of this flow that is cleanly reversible.

## Conventions and reversibility

- **Validation errors use the batch envelope**, not the POS one:
  `{message, errorType, errorMsgs[], failed[]}`. `failed[]` names *which* submitted entity broke
  and why (`"attributeCategoryId is required"`). Correct only the failing entities and resubmit —
  do not blindly resend the whole batch.
- **Products and SKUs cannot be deleted.** There is no delete operation for either. Reversal is
  *deactivation*: set the product inactive, and it stays retrievable through the product-list
  `active=FALSE|ALL` filter. Store prices, discounts, custom subcategories and subcategory
  exclusions *do* have DELETE operations.
- **Custom Subcategories are in limited release** and require the "Catalog Management — Custom
  Subcategories" feature flag on the organization. There is no way to detect the flag from the
  contract; if the endpoints reject you, that is why.
- Catalog ids are camelCase (`productId`, `variantId`, `brandId`); the dispensary POS surface uses
  snake_case (`product_id`). Nothing in either contract states that they correspond — see
  `data-model/treez-data-model.yml`.
