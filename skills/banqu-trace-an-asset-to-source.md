---
generated: '2026-08-06'
method: generated
name: Trace an asset back to source
description: >-
  Walk a BanQu asset or transfer backwards through its chain of custody to the originating suppliers,
  transformations, locations and transactions - the traceability query behind EUDR and
  responsible-sourcing evidence.
api: openapi/banqu-openapi-original.json
operations:
- 'GET /assets'
- 'GET /assets/{assetId}'
- 'GET /assets/{assetId}/sources'
- 'GET /assets/{assetId}/history'
- 'GET /assets/{assetId}/holders'
- 'GET /transfers/{transferId}/sources'
- 'GET /transactions/{transactionId}/operations'
- 'GET /shared-profile/{ownerId}'
- 'GET /shared-profile/{ownerId}/geo-data'
source: >-
  Grounded in openapi/banqu-openapi-original.json (OpenAPI 3.0.3, harvested verbatim from
  https://banqu.app/api/v1/schema). The spec declares NO operationIds, so every step cites the
  verified HTTP method and path. Entity graph per data-model/banqu-data-model.yml.
---

# Trace an asset back to source

This is BanQu's reason to exist: given a batch of finished or aggregated product, prove which farms,
plots, collectors or waste pickers it came from.

## Base URL

`https://banqu.app/api/v1` — authenticate first with
`skills/banqu-authenticate-and-select-account.md`.

## Steps

1. **Find the asset** — `GET /assets` with `search`, `offset` and `limit` (default 20), or
   `GET /assets/{assetId}` if you already hold the id. The `Asset` schema carries the code,
   description, unit of measure, prices and owner.
2. **Pull the immediate sources** — `GET /assets/{assetId}/sources`. This returns the transfer
   operations that supplied the current holding. Each is an `AssetTransferOperation` with
   `senderId`, `receiverId`, `assetId`, `transactionId`, quantity, `effectiveDate` and
   `effectiveLocation`.
3. **Recurse one level up per transfer** — for every operation returned, call
   `GET /transfers/{transferId}/sources`. Repeat until a transfer has no sources; that is an
   origination event (a deposit or a first-mile collection). Keep a visited set — transformations can
   fan in from many inputs and you do not want to walk the same operation twice.
4. **Handle transformations explicitly.** An `AssetTransformationOperation` consumes input assets into
   a new output asset, so the asset id changes as you cross it. When the chain crosses a
   transformation, switch back to `GET /assets/{newAssetId}/sources` for the input side rather than
   continuing on transfer ids alone.
5. **Get the full movement log** — `GET /assets/{assetId}/history` returns `AssetHistory`, the
   chronological record. `GET /assets/{assetId}/holders` returns who holds quantity right now.
6. **Resolve the counterparties to real suppliers** — for each distinct `senderId`, call
   `GET /shared-profile/{ownerId}` for the supplier profile shared to you, and
   `GET /shared-profile/{ownerId}/geo-data` for the GeoJSON feature collection. **The geo-data call is
   the plot-level evidence** an EUDR due-diligence statement needs.
7. **Attach the commercial context** — `GET /transactions/{transactionId}/operations` expands a
   transaction into all of its transfer operations, and
   `GET /transactions/{transactionId}/history` gives its change log.

## Notes

- Timestamps everywhere are **numeric epoch milliseconds**, not RFC 3339 strings.
- Locations are GeoJSON (`GeoJsonFeatureCollection` / `GeoJsonGeometryObject`) via
  `EffectiveLocation`, so plot polygons drop straight into a mapping stack.
- Pagination is `offset` + `limit`. There is no total-count field and no next-link, so read until a
  short page comes back.
- All of these are `GET`s and therefore safe to retry — which matters, because BanQu documents no
  rate limit and no `Retry-After` header. Back off on `429` anyway; the response component exists
  even though no operation references it.
- If you only have visibility through a share rather than ownership, the equivalent read paths are
  `GET /assets/shares/{shareId}/transfers` and
  `GET /assets/shares/{shareId}/transfers/{transferId}/sources`.

## Errors

- `403` — the counterparty has not shared that profile or asset with your account. Chain-of-custody
  visibility is share-scoped, so a partial trace is a normal outcome, not a bug.
- `404` — the asset, transfer or transaction id is not visible to the selected account. Re-check
  which account your token was minted for.
- See `errors/banqu-problem-types.yml`; 4xx bodies have no declared schema.
