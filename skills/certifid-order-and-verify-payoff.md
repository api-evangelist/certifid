---
name: certifid-order-and-verify-payoff
description: Order a mortgage payoff statement from a lender through CertifID and verify a payoff disbursement, covering lender lookup, order lifecycle, statement upload, verification fields, and the quote and disbursement PDFs.
generated: '2026-08-09'
method: generated
source: openapi/certifid-v2-apis-openapi.json
api: CertifID V2 APIs
base_url: https://api.certifid.com
operations:
  - PayoffOrderingGetAvailablePayoffOrderLenders
  - LendersSearchLenders
  - PayoffOrderingCreatePayoffOrder
  - PayoffOrderingGetPayoffOrderByIdById
  - PayoffOrderingUpdatePayoffOrder
  - PayoffOrderingCancelPayoffOrder
  - GetPayoffOrderingGetCertificateQuoteCertificateByPayoffOrderIdQuoteId
  - DisbursementsCreateDisbursement
  - DisbursementsEditDisbursement
  - DisbursementsGetDisbursementByIdById
  - DisbursementsSearchDisbursements
  - DisbursementsUploadDisbursementDocument
  - DisbursementsGetDisbursementVerificationFieldsFromStatement
  - DisbursementsGetDisbursementStatementsByDisbursementIdById
  - DisbursementsGetLoanType
  - DisbursementsGetDisbursementCancelationReasons
  - PostDisbursementsGetPdfByDisbursementIdByDisbursementId
---

# Order and verify a mortgage payoff

Two adjacent surfaces that together cover CertifID's payoff products. **PayoffOrdering**
orders a payoff statement from a lender. **Disbursements** (called *PayoffProtect* in the
schemas) verifies the payoff before funds move. They are the largest part of the API — 16
of 57 operations.

> Naming warning: the paths say `Disbursements`, the schemas say `PayoffProtect`, and the
> product pages say "Order & Verify: Payoffs". These are the same thing.

## Auth and scopes

OAuth 2.0 bearer token, audience `https://api.certifid.com`. Requires `CertifIDUser` and
`PayoffProtectProduct` (the second-most-required scope in the API, on 12 operations).
Per-resource reads use `ResourceAccessToken|read:payofforder|payoffOrderId|PayoffOrder`
and `ResourceAccessToken|read:disbursements|id|DisbursementVerification`.

## Method warning

This cluster is RPC-shaped: **reads are `POST`, not `GET`.**
`POST /api/v1/Disbursements/GetDisbursementById/{id}` is a read. Do not assume HTTP-method
semantics anywhere in the Disbursements or Lenders groups, and do not let a caching layer
treat these as safe.

## Ordering a payoff

1. **Find the lender** — `GET /api/v1/PayoffOrdering/GetAvailablePayoffOrderLenders`
   (**PayoffOrderingGetAvailablePayoffOrderLenders**) lists lenders you can order from.
   `POST /api/v1/Lenders/SearchLenders` (**LendersSearchLenders**) is a *partial* name
   search. `POST /api/v1/Lenders/GetLenderLogoById` retrieves a logo.
2. **Create** — `POST /api/v1/PayoffOrdering/CreatePayoffOrder`
   (**PayoffOrderingCreatePayoffOrder**).
3. **Read** — `POST`… no: `GET /api/v1/PayoffOrdering/GetPayoffOrderById/{id}`
   (**PayoffOrderingGetPayoffOrderByIdById**) — this one genuinely is a `GET`.
4. **Update / cancel** — `POST /api/v1/PayoffOrdering/UpdatePayoffOrder` and
   `POST /api/v1/PayoffOrdering/CancelPayoffOrder`.
5. **Quote PDF** —
   `GET /api/v1/PayoffOrdering/GetCertificate/{payoffOrderId}/quote/{quoteId}/Certificate`.
   A quote is addressed by the **order and quote pair**; it is not independently
   addressable.

`PayoffOrder` carries `parentOrderId` / `childOrderId`, so orders form a tree — an order
can be split or superseded. It is also the only entity in the API with an event history
(`PayoffOrderEvent`), so use it rather than polling a status field where you can.

## Verifying a disbursement

1. **Reference data first** — `POST /api/v1/Disbursements/GetLoanType` and
   `POST /api/v1/Disbursements/GetDisbursementCancelationReasons` return the enumerations
   you need before you can populate a create.
2. **Create / edit** — `POST /api/v1/Disbursements/CreateDisbursement`
   (**DisbursementsCreateDisbursement**) and
   `POST /api/v1/Disbursements/EditDisbursement`.
3. **Upload the statement** — `POST /api/v1/Disbursements/UploadDisbursementDocument`
   saves the payoff statement as a **PDF** in CertifID's system.
4. **Extract verification fields** —
   `POST /api/v1/Disbursements/GetDisbursementVerificationFieldsFromStatement` pulls the
   verifiable fields out of the uploaded statement. This is the step that turns a
   document into checkable data.
5. **List statements** —
   `POST /api/v1/Disbursements/GetDisbursementStatementsByDisbursementId/{id}`.
6. **Read / search** — `POST /api/v1/Disbursements/GetDisbursementById/{id}` and
   `POST /api/v1/Disbursements/SearchDisbursements` (paged — see below).
7. **Certificate** — `POST /api/v1/Disbursements/getPdfByDisbursementId/{disbursementId}`
   returns the confirmation PDF.

## Pagination

`SearchDisbursements` and `AccountVerifications/Search` return a `PagedData` envelope:
`items`, `pageSize`, `zeroBasedCurrentPage`, `totalItems`, `totalPages`.

**Paging is zero-based and travels in the POST body**, not the query string. Requesting
page `1` returns the *second* page. There is no cursor pagination and no `Link` header,
and because the parameters are in the body, a page is not a bookmarkable URL.

## Safety

`CreateDisbursement` and `CreatePayoffOrder` bind real money movement in a closing.
**Neither is idempotent and neither accepts an `Idempotency-Key`.** A retried timeout can
duplicate a disbursement. Reconcile with `SearchDisbursements` before re-issuing any
create. Treat both as consequential actions requiring explicit confirmation in any agent
context.

A `403` here means the company lacks `PayoffProtectProduct` entitlement, or the consumer
cannot access that specific disbursement — both are documented causes and neither is
fixed by retrying.
