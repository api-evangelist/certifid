---
name: certifid-verify-account-and-business
description: Create, search, retrieve and cancel CertifID account verifications to confirm bank account and business ownership for a party to a real-estate transaction.
generated: '2026-08-09'
method: generated
source: openapi/certifid-v2-apis-openapi.json
api: CertifID V2 APIs
base_url: https://api.certifid.com
operations:
  - PostAccountVerifications
  - AccountVerificationsSearch
  - GetAccountVerificationsById
  - AccountVerificationsCancelById
  - GetBankByAbaRoutingNumber
  - GetBankWiringDetailsByAbaRoutingNumber
---

# Verify a bank account or a business

The account-verification surface confirms that a bank account or a business is genuinely
owned by the party claiming it. It backs CertifID's "Verify: Businesses" and bank-account
verification products and maps to the **Payoff / Business Verifications** status
component.

## Auth and scopes

OAuth 2.0 bearer token, audience `https://api.certifid.com`. Requires `CertifIDUser`;
some paths also require `RequestCreator`.

## Building the request

`POST /api/v1/AccountVerifications` (**PostAccountVerifications**) — "Create a new
account verification."

The request model (`CreateAccountVerificationRequestModel`) composes five sub-objects,
and getting this shape right is most of the work:

- **`party`** — polymorphic. An **individual** carries a person name, an address and an
  individual tax id; a **business** carries a business registration, a business tax id and
  an address. This is the most reused sub-graph in the API.
- **`account`** — the bank account being verified.
- **`transaction`** — carries a `PaymentRail` and either a `B2BWireContext` or a
  `PayoffContext`. The payoff context itself nests a disbursement party, an underwriter
  and an address.
- **`useCase`** — from the `AccountVerificationUseCase` enum.
- **`externalReference`** — your own correlation key.

**Set `externalReferenceId`.** It is the only client-owned field in the model, it is
filterable on search, and it is how you find your record again after a failed call. It is
*not* an idempotency key — nothing in the contract says a replay with the same value
returns the original verification instead of creating a second one.

## Reading back

- `GET /api/v1/AccountVerifications/{id}` (**GetAccountVerificationsById**).
- `POST /api/v1/AccountVerifications/Search` (**AccountVerificationsSearch**) — searches
  verifications for the authenticated tenant.

Search accepts `pageSize`, `zeroBasedPage`, `sortFields`, `query`, `status`, `useCase`,
`partyType`, `externalReferenceId`, and an inclusive `createdGte`/`createdLte` date range.
Paging is **zero-based** and lives in the request body. The response is a `PagedData`
envelope (`items`, `pageSize`, `zeroBasedCurrentPage`, `totalItems`, `totalPages`).

Status values come from the `AccountVerificationStatus` enum.

## Cancelling

`POST /api/v1/AccountVerifications/{id}/Cancel` (**AccountVerificationsCancelById**).

## Supporting bank lookup

`GET /api/v1/Bank/{abaRoutingNumber}` and
`GET /api/v1/Bank/{abaRoutingNumber}/WiringDetails` resolve an ABA/routing number to bank
details and wiring details. These are reference-data reads, not tenant-scoped, and are
useful for validating input before you create a verification.

## Reconciling instead of retrying

The create is not idempotent and accepts no `Idempotency-Key`. If it times out, call
`AccountVerificationsSearch` filtered by your `externalReferenceId` and the
`createdGte`/`createdLte` window before issuing another create. That search is the only
deduplication mechanism the API gives you.

No webhook exists — poll `GetAccountVerificationsById` with backoff. No rate limit is
documented and no `429` is declared anywhere, so keep polling conservative.
