---
generated: '2026-08-06'
method: generated
name: Authenticate and select a BanQu account
description: >-
  Mint a BanQu credential correctly - discover which accounts your identity can act as, choose one,
  and get either a short-lived session token or a persistent API token for automation.
api: openapi/banqu-openapi-original.json
operations:
- 'GET /auth/accounts'
- 'POST /auth/accounts/{accountId}/token'
- 'POST /auth/api-tokens'
- 'GET /auth/api-tokens'
- 'DELETE /auth/api-tokens/{id}'
- 'GET /orgs/current/capabilities'
source: >-
  Grounded in openapi/banqu-openapi-original.json (OpenAPI 3.0.3, harvested verbatim from
  https://banqu.app/api/v1/schema). The spec declares NO operationIds, so every step below cites the
  verified HTTP method and path instead. Auth per authentication/banqu-authentication.yml, errors per
  errors/banqu-problem-types.yml.
---

# Authenticate and select a BanQu account

The single fact that trips up every BanQu integration: **the account you act as is chosen when the
token is minted, not per request.** One human identity can be a personal account plus several
organizations. Get this wrong and every subsequent call returns data for the wrong org, or 403.

## Base URL

`https://banqu.app/api/v1`

## Steps

1. **List the accounts your identity can act as** — `GET /auth/accounts`. Returns an array of
   `AccountInfo`: `id` (a fixed 32-character string), `displayName`, `logo`, and `isOrg`. Pick the
   account whose data you intend to read or write.
2. **Mint a session token for that account** — `POST /auth/accounts/{accountId}/token`. Returns
   `AuthTokens`: `token`, `tokenExpires`, `refreshToken`, `refreshTokenExpires`. This is the
   short-lived path; refresh before `tokenExpires` (epoch **milliseconds**, not seconds).
3. **For unattended automation, mint a persistent API token instead** —
   `POST /auth/api-tokens` with an `ApiTokenCreationParameters` body: a `title` plus the calling
   **user's current password**. The response body is the JWT, as a bare JSON string.
   > BanQu states plainly: *"Token will be written to the response body only once and will not be
   > stored within the BanQu system."* If you lose it, you mint a new one — there is no recovery.
4. **Send the token on every call.** The security scheme is HTTP Bearer with `bearerFormat: JWT`, so
   `Authorization: Bearer <token>`. Note the contract conflict recorded in
   `authentication/banqu-authentication.yml`: the `AuthTokens` schema describes the token as going in
   an `X-BQ-Token` header. Both statements are in the published spec. If Bearer 401s, try
   `X-BQ-Token`.
5. **Confirm what the token can actually do** — `GET /orgs/current/capabilities`. Returns
   `OrgCapabilities`: a map of capability name to a set of `create | read | update | delete`. Do this
   before your first write; it is the only way to discover your effective permissions.
6. **Inventory and rotate** — `GET /auth/api-tokens` lists existing tokens (`id`, `title`, `created`,
   `expires`); `DELETE /auth/api-tokens/{id}` revokes one. Rotate by minting the new token, cutting
   traffic over, then deleting the old id.

## Notes

- **There are no OAuth scopes.** BanQu has no OAuth 2.0 and no OIDC. A token inherits everything its
  underlying user can do. The only way to least-privilege an integration is to create a dedicated
  BanQu user with a narrow org role (`POST /orgs/current/roles`) and mint the token as that user.
- **API tokens require a human password**, so there is no pure machine-to-machine credential. Treat
  the service user's password as a secret of equal blast radius to the token.
- Signup is invite-token based (`POST /auth/signup`, `GET|POST /auth/signup/{token}`). There is no
  public self-serve developer signup — access comes through a BanQu customer relationship.

## Errors

- `401` — "User is not authenticated or session has expired." Re-mint or refresh.
- `403` — "Not authorized to access selected resource." Usually the wrong account was selected in
  step 2, or the org role lacks the capability. Check step 5.
- `422` on `POST /auth/api-tokens` — the password or title failed validation.
- Error responses carry **no body schema** in the spec; see `errors/banqu-problem-types.yml`. This
  API is not RFC 9457.
