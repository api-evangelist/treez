---
name: treez-ecommerce-order-intake
description: Place an online or delivery order into a Treez dispensary POS - resolve or create the customer, price the basket with a non-committing preview, then create the ticket and track it to completion.
api: openapi/treez-dispensary-openapi.json
generated: '2026-08-30'
method: generated
source: https://code.treez.io/recipes/creating-an-order-with-postcreate-ticket
operations:
  - dispensary-customer-get-by-email
  - dispensary-customer-get-by-phone
  - dispensary-customer-create
  - ticket-preview
  - create-ticket
  - ticket-get-by-id
  - ticket-get-by-order-number
  - ticket-update
---

# Push an ecommerce order into Treez

This is the flow every menu, delivery and marketplace integration runs. It is a **write** flow
with money and state-regulated inventory on the other side, so read the guardrails at the bottom
before you run it unattended.

`{dispensary}` throughout is the tenant slug — the subdomain of the customer's own
`<tenant>.treez.io` URL. Base: `https://api-prod.treez.io/dispensary/v3/{dispensary}`.

## Steps

1. **Resolve the customer before creating one.**
   Call `dispensary-customer-get-by-email`
   (`GET /{dispensary}/customer/email/{email}`), or `dispensary-customer-get-by-phone`
   (`GET /{dispensary}/customer/phone/{phone_number}`) when you only hold a phone number.
   A `404` with `resultReason: CUSTOMER_DOESNT_EXIST` is the expected miss — it is not an error
   condition to retry.

2. **Create the customer only on a confirmed miss.**
   `dispensary-customer-create` (`POST /{dispensary}/customer/detailcustomer`).
   `first_name` is required — omitting it returns `400 CUSTOMER_FIRST_NAME_BLANK`. Carry your own
   identifier in `external_id` so the next sync can match without a search.
   **This POST is not idempotent and there is no Idempotency-Key.** If it times out, look the
   customer up again before retrying, or you will create a duplicate that later needs
   `dispensary-customer-merge`.

3. **Price the basket without committing it.**
   `ticket-preview` (`POST /{dispensary}/ticket/previewticket`) returns the fully priced ticket —
   taxes, automatic discounts, subtotal, total — with `order_status: PREVIEW`. Nothing is created.
   This is the only rehearsal surface Treez publishes, and it is the right place to catch
   `TICKET_FORMAT_ERROR` and `PRODUCT_NOT_FOUND` before you take the customer's money.

4. **Show the previewed total to the buyer, then create the ticket.**
   `create-ticket` (`POST /{dispensary}/ticket/detailticket`). The response carries `ticket_id`
   (GUID) and `order_number` (a 6-character customer-facing code). **Persist both.** They are not
   interchangeable — passing one where the other is expected returns `400 TICKET_NOT_FOUND`.

5. **Track the order.**
   `ticket-get-by-id` (`GET /{dispensary}/ticket/{ticket_id}`) for your own polling;
   `ticket-get-by-order-number` (`GET /{dispensary}/ticket/ordernumber/{order_number}`) when a
   human quotes the short code. Watch `order_status` (e.g. `AWAITING_PROCESSING`) and
   `payment_status` (e.g. `UNPAID`) — they move independently.

6. **Amend or cancel through `ticket-update`**
   (`PUT /{dispensary}/ticket/update/{ticket_id}`). An illegal transition returns
   `400 INVALID_TICKET_STATUS`, so read the current status first.

## Guardrails

- **No idempotency anywhere.** No `Idempotency-Key` header exists on any Treez operation. Treat
  every POST as fire-once. On a timeout, *read back* (by `external_order_number` or customer
  history) before retrying.
- **No request-id header.** There is nothing to quote to `api-support@treez.io` when a call goes
  wrong. Log the full request and the `resultReason` yourself.
- **Reversal is possible, the window is not published.** A refund is modelled as a *new* ticket
  linked to the original through `original_ticket_id` / `refund_ticket_ids`; cancellation is an
  `order_status` transition to `CANCELED`. Treez publishes **no deadline** for either. Do not tell
  a user an order can still be cancelled — read the ticket and find out.
- **Errors are in the body, not just the status.** The envelope is
  `{resultCode, resultReason, resultDetail, data}` and `resultCode: FAIL` is the discriminator.
  See `errors/treez-problem-types.yml`.
- **No 429 and no rate-limit headers are published.** Pace yourself conservatively; you will get
  no runtime backoff signal.
