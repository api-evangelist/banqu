---
generated: '2026-08-06'
method: generated
name: Collect and approve supplier form data
description: >-
  Assign a BanQu form to suppliers, submit entries (including file attachments), and move each entry
  through the review, approval and denial lifecycle - the mechanism behind due-diligence and
  compliance evidence collection.
api: openapi/banqu-openapi-original.json
operations:
- 'GET /forms'
- 'GET /forms/{formId}'
- 'POST /forms/{formId}/assign'
- 'GET /forms/{formId}/data-entries'
- 'POST /forms/{formId}/data-entries'
- 'PATCH /forms/{formId}/data-entries/{entryId}'
- 'POST /forms/{formId}/data-entries/{entryId}/submit'
- 'POST /forms/{formId}/data-entries/{entryId}/review'
- 'POST /forms/{formId}/data-entries/{entryId}/approval'
- 'POST /forms/{formId}/data-entries/{entryId}/denial'
- 'GET /aws/upload-url'
source: >-
  Grounded in openapi/banqu-openapi-original.json (OpenAPI 3.0.3, harvested verbatim from
  https://banqu.app/api/v1/schema). The spec declares NO operationIds, so every step cites the
  verified HTTP method and path. Conventions per conventions/banqu-conventions.yml.
---

# Collect and approve supplier form data

Forms are how BanQu captures everything that is not a quantity: certifications, plot boundaries,
labour attestations, regenerative-practice questionnaires, EPR declarations.

## Base URL

`https://banqu.app/api/v1` — authenticate first with
`skills/banqu-authenticate-and-select-account.md`.

## Steps

1. **Find the form definition** — `GET /forms`, then `GET /forms/{formId}` for the full `Form`:
   groups, sections, and typed fields (`FormField`, `FormFieldWithItems`, `CheckboxField`,
   `PersonNameField`, with `FormFieldImportance` and `FormElementVisibility`).
   `GET /forms/fragments` returns reusable field fragments; `GET /forms/{formId}/editable` tells you
   whether the definition can still be changed.
2. **Assign it** — `POST /forms/{formId}/assign` to push the form to the connections that must
   complete it.
3. **Create an entry** — `POST /forms/{formId}/data-entries` with a `FormDataEntry`. Update a draft
   with `PATCH /forms/{formId}/data-entries/{entryId}`.
4. **Attach files** — `GET /aws/upload-url` mints a pre-signed upload URL; `PUT` the bytes to that
   URL directly, then reference the resulting `Attachment` in the entry. Do not try to post file
   bytes through the form endpoints.
5. **Submit** — `POST /forms/{formId}/data-entries/{entryId}/submit`. Withdraw a submission with
   `.../withdraw`.
6. **Route it for review** — `POST .../review` records a review;
   `POST .../responders` (or `PUT` to replace the whole set) assigns who must respond, and
   `DELETE .../responders/{responderId}` removes one.
7. **Decide** — `POST .../approval` to approve, `POST .../denial` to deny. Both are reversible:
   `DELETE .../approval` revokes an approval and `DELETE .../denial` revokes a denial.
   `POST .../reject` sends the entry back to the submitter.
8. **Read the results** — `GET /forms/{formId}/data-entries` with `offset`, `limit`, `sortBy` and
   `search`. Set `formId` to the literal string `all` to read entries across every form at once —
   that is the reporting query.
9. **Bulk import instead of hand entry** — if suppliers send spreadsheets, register a Data Processor
   (`POST /data-processors` with a CSV or XLSX parser plus mapping steps) and run it with
   `POST /data-processors/{processorId}/process`. Dry-run it first with
   `POST /data-processors/{processorId}/validate`, and stop a runaway job with `.../stop`.

## Notes

- **Natural-key addressing.** `entryId` accepts either a BanQu id **or** a double-pipe-separated
  (`||`) list of the globally primary field values, in the order the fields appear in the UI — e.g.
  `value1||value 2||value-3`. This is the closest thing BanQu offers to an idempotent upsert: address
  an entry by its business key and you will not create a duplicate on retry. It does **not** work
  when `formId` is `all`.
- **Set `preventNotify=true`** on bulk writes to suppress notification side effects.
- Entries and forms are soft-deleted: `DELETE` then `POST .../restore`. Duplicate an entry as a
  template with `POST .../duplicate`.
- Timestamps are numeric epoch milliseconds.

## Errors

- `400` — required fields missing, or a `||` natural key with the wrong number of segments.
- `403` — the org role lacks the capability, or the entry belongs to a form not shared with your
  account.
- `404` — unknown `formId`/`entryId` for the selected account; also returned when a natural key
  matches nothing.
- `422` — field-level validation failed, or a Validation Workflow attached to this write action
  rejected it. Check `GET /validation-workflows` to see what custom rules the org has installed.
- See `errors/banqu-problem-types.yml`; 4xx bodies carry no declared schema.
