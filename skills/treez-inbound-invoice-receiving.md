---
name: treez-inbound-invoice-receiving
description: Receive a delivery from a distributor into a Treez dispensary - sync METRC packages, create and pay the invoice, and correct it with vendor credits or inventory adjustments.
api: openapi/treez-dispensary-openapi.json
generated: '2026-08-30'
method: generated
source: https://code.treez.io/reference/invoice
operations:
  - dispensary-distributor-get-all
  - payment-terms-get-all
  - sync-metrc-packages
  - invoice-create
  - invoice-apply-payment
  - invoice-apply-credit-1
  - invoice-apply-credit
  - put_dispensary-inventory-adjustment
  - get_dispensary-inventory-reasons
  - put_dispensary-inventory-move
---

# Receive an inbound delivery into Treez

This is the money-and-compliance flow: it moves state-tracked cannabis inventory onto the shelf
and creates a payable. Base: `https://api-prod.treez.io/dispensary/v3/{dispensary}`.

## Steps

1. **Resolve the distributor.** `dispensary-distributor-get-all`
   (`GET /{dispensary}/distributor/list`). If the vendor is new,
   `distributor-create` (`POST /{dispensary}/distributor`) first.

2. **Resolve payment terms.** `payment-terms-get-all` (`GET /{dispensary}/paymentTerm/list`), or
   create one with `payment-terms-get-all-1` (`POST /{dispensary}/invoice/paymentTerm`).

3. **Sync the state track-and-trace manifest.** `sync-metrc-packages`
   (`POST /{dispensary}/trace/packages/metrcsync`) — Treez's published description is "Retrieves
   available METRC packages for invoice creation". Run this *before* building the invoice so the
   lines bind to real, state-visible packages rather than to hand-typed labels. This is the one
   place in the Treez contract where the domain standard is named in the path namespace, and it is
   what makes the received inventory reconcilable against the state system.

4. **Create the invoice.** `invoice-create` (`POST /{dispensary}/invoice`), with lines referencing
   the packages from step 3.

5. **Apply payment.** `invoice-apply-payment` (`POST /{dispensary}/invoice/payment`).

6. **Attach the paperwork** if you hold it — `invoice-apply-payment-1`
   (`POST /{dispensary}/attachment`, multipart) or `invoice-apply-payment-1-1`
   (`POST /{dispensary}/invoice/attachment/binary`).

## Corrections

Nothing here is deletable. Every correction is a **compensating action**, which is the right model
for a regulated ledger but means you must plan for it rather than assume an undo.

- **Wrong amount owed** → create a vendor credit with `invoice-apply-credit-1`
  (`POST /{dispensary}/invoice/credit/create`). Treez states explicitly that this "only creates the
  credit — it does not apply it to a specific invoice". Then apply it with `invoice-apply-credit`
  (`POST /{dispensary}/invoice/credit/apply`). Two calls, and the first one on its own leaves an
  unapplied credit floating.
- **Wrong invoice detail** → `invoice-modify` (PUT) or `invoice-modify-patch` (PATCH) on
  `/{dispensary}/invoice`.
- **Wrong quantity received** → `put_dispensary-inventory-adjustment`
  (`PUT /{dispensary}/inventory/adjustment`). Pull the allowed reason codes first with
  `get_dispensary-inventory-reasons` (`GET /{dispensary}/inventory/reasons`) — the reason is part
  of the audit record, not a free-text note.
- **Wrong shelf** → `put_dispensary-inventory-move` (`PUT /{dispensary}/inventory/move`) moves
  inventory between locations within the store.
- **Merge Batches** (`post_dispensary-inventory-merge`) has **no published un-merge.** Treat it as
  one-way.

## Guardrails

- **No idempotency key exists on any of these operations.** A retried `invoice-create` or
  `invoice-apply-payment` creates a second record. On a timeout, read back with
  `invoice-get-by-number` or `invoice-get-by-date-range` before retrying.
- **Date-range invoice queries are capped at 30 days per request** — a wider range returns
  `400 RESPONSE_LIMIT_EXCEEDS`.
- **No reversal window is published for anything here.** Treez documents the reversal *paths*
  above but states no deadline for any of them. Never tell a user a payment can still be credited
  without reading current state first.
- An agent should not run steps 4–6 unattended without a human confirmation step. These are
  `acting` operations with financial and regulatory consequence, and the API offers neither a
  dry-run nor a retry-safe write.
