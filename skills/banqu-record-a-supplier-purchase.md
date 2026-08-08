---
generated: '2026-08-06'
method: generated
name: Record a supplier purchase
description: >-
  Onboard or resolve a supplier connection, record the purchase as a BanQu transaction with its asset
  transfers and payment destination, apply any deductions, and finalize it.
api: openapi/banqu-openapi-original.json
operations:
- 'GET /connections'
- 'POST /connections'
- 'POST /connections/filter'
- 'POST /connections/{connectionId}/resend'
- 'GET /assets'
- 'POST /transactions'
- 'POST /transactions/batch'
- 'POST /assets/{assetId}/transfers'
- 'POST /payment-adjustments'
- 'POST /transactions/{transactionId}/finalize'
- 'GET /transactions/{transactionId}'
source: >-
  Grounded in openapi/banqu-openapi-original.json (OpenAPI 3.0.3, harvested verbatim from
  https://banqu.app/api/v1/schema). The spec declares NO operationIds, so every step cites the
  verified HTTP method and path. Conventions per conventions/banqu-conventions.yml.
---

# Record a supplier purchase

The core write flow: a buyer receives goods from a farmer, collector or supplier, and the exchange is
written to the ledger with the payment leg attached.

## Base URL

`https://banqu.app/api/v1` — authenticate first with
`skills/banqu-authenticate-and-select-account.md`, and confirm you hold `create` on the relevant
capabilities via `GET /orgs/current/capabilities`.

## Steps

1. **Resolve the supplier** — `GET /connections` with `search`, or `POST /connections/filter` for a
   structured query. A `connectionId` may be any of: a userId, an orgId, an inviteId, a **verified
   email**, a **verified phoneNumber**, a userName, or your own **externalId**. That last one is the
   integration lever — if you push your ERP's supplier code as `externalId`, you never need to store
   BanQu ids.
2. **Create the connection if it does not exist** — `POST /connections` with a
   `ConnectionCreatePayload`. If the invite goes unanswered,
   `POST /connections/{connectionId}/resend`. A supplier accepts via
   `POST /connections/{connectionId}/accept`.
3. **Identify the asset** — `GET /assets` (filter with `search` / `codes`), or `POST /assets` if the
   commodity is not yet defined. Note the unit of measure on the `Asset`; quantities are expressed in
   it.
4. **Record the exchange.** Two shapes, pick one:
   - **Transaction-first (preferred for buy/sell)** — `POST /transactions` with a
     `CreateTransactionRequest`. This models the economic event as an `ExchangeTransaction`,
     including the payment leg and its `PaymentDestination`.
   - **Transfer-first** — `POST /assets/{assetId}/transfers` with an `AssetTransferRequest` when you
     are only moving quantity, or `POST /assets/{assetId}/transfers/batch` for many at once.
5. **Set the payment destination.** `PaymentDestination` is either a BanQu agency payout or a
   mobile-money provider. The enum published in the spec is: `banqu-agency`, `mtn-zambia`,
   `6dot50pay-zaf`, `mtn-uganda`, `tigo-tanzania`, `celbux-zaf`, `airtel`, `zamtel`, `vodacom`.
   Do not invent a provider string — the enum is closed.
6. **Apply deductions or bonuses** — `POST /payment-adjustments` with a `CurrencyPaymentAdjustment`
   (absolute) or a `PercentagePaymentAdjustment`. List them back with `GET /payment-adjustments`.
7. **Finalize** — `POST /transactions/{transactionId}/finalize` promotes a draft to a settled
   transaction. Verify with `GET /transactions/{transactionId}` and
   `GET /transactions/{transactionId}/operations`.
8. **On the receiving side**, the counterparty confirms with
   `POST /transfers/{transferId}/confirmReceipt` or declines with
   `POST /transfers/{transferId}/rejectReceipt`. Unclaimed last-mile transfers use the claim-token
   flow: `POST /agent/transfers/{claimToken}/claim` then
   `POST /agent/transfers/{claimToken}/finalize`, with
   `POST /transfers/{transferId}/resendClaimToken` if the token was lost.

## Notes

- **There is no idempotency key.** Not on `POST /transactions`, not on
  `POST /transactions/batch`, not on `POST /assets/{assetId}/transfers/batch`. A retried POST after a
  timeout creates a **second** transaction. Before retrying, re-read `GET /transactions` filtered by
  `codes` and `transactionKind` and reconcile on your own reference field. This is the number-one
  operational hazard in this API.
- **Bulk imports should set `preventNotify=true`** — the reusable query parameter suppresses the
  notification side effects, so you do not page every supplier during a backfill.
- Mistakes are recoverable: `DELETE /transactions/{transactionId}` soft-deletes, and most resources
  have a `POST .../restore` counterpart. `RevertTransaction` exists in the schema set as a
  transaction kind.
- `transactionKind` values are `buy`, `sell`, `transfer`, `deposit`, `transformation`,
  `draft-locked`, `draft-unlocked`.

## Errors

- `400` — required parameters missing from the body.
- `403` — the org role lacks `create` on transactions or transfers.
- `409` — conflict with the current state of the target resource; on a finalize this usually means
  the transaction is already finalized. Re-read before retrying.
- `422` — the request was well formed but could not be processed (e.g. quantity exceeds the holding,
  or a payment destination the supplier has not configured).
- See `errors/banqu-problem-types.yml`; 4xx bodies carry no declared schema.
