---
name: certifid-collect-and-confirm-wiring-instructions
description: Collect a seller's or payee's wiring instructions through CertifID and confirm previously supplied instructions before disbursing, then retrieve the PDF certificate for each.
generated: '2026-08-09'
method: generated
source: openapi/certifid-v2-apis-openapi.json
api: CertifID V2 APIs
base_url: https://api.certifid.com
operations:
  - PostWiringInstructionsCollect
  - GetWiringInstructionsCollectByRequestId
  - GetWiringInstructionsCollectCertificateByRequestId
  - PostWiringInstructionsConfirm
  - GetWiringInstructionsConfirmByRequestId
  - GetWiringInstructionsConfirmCertificateByRequestId
---

# Collect and confirm wiring instructions

Two sibling flows that mirror the Send flow in the opposite direction. **Collect** asks a
seller or payee to supply their wiring instructions over a verified channel. **Confirm**
validates instructions you already hold before money moves.

Both use the same request envelope and status enum as Send, and both terminate in a PDF
certificate. Pick Collect when you do not yet have the instructions and Confirm when you
do.

> Operation ids come from `overlays/certifid-certifid-v2-apis-overlay.yaml`; paths and
> methods are verbatim from the live specification.

## Auth and scopes

OAuth 2.0 bearer token, audience `https://api.certifid.com`. Requires `CertifIDUser` and
`RequestProduct`; per-request reads carry
`ResourceAccessToken|read:requests|requestId|CollectRequest` or
`...|ConfirmRequest` respectively.

## Collect

1. **Initiate** — `POST /api/v1/WiringInstructions/Collect`
   (**PostWiringInstructionsCollect**): "Initiate a new request to request a seller's
   wiring instructions."
2. **Poll** — `GET /api/v1/WiringInstructions/Collect/{requestId}`
   (**GetWiringInstructionsCollectByRequestId**).
3. **Certificate** — `GET /api/v1/WiringInstructions/Collect/{requestId}/Certificate`
   (**GetWiringInstructionsCollectCertificateByRequestId**), returns a PDF.

## Confirm

1. **Initiate** — `POST /api/v1/WiringInstructions/Confirm`
   (**PostWiringInstructionsConfirm**): "Initiate a new request to Confirm a seller's
   wiring instructions."
2. **Poll** — `GET /api/v1/WiringInstructions/Confirm/{requestId}`
   (**GetWiringInstructionsConfirmByRequestId**).
3. **Certificate** — `GET /api/v1/WiringInstructions/Confirm/{requestId}/Certificate`
   (**GetWiringInstructionsConfirmCertificateByRequestId**), returns a PDF.

## Rules that apply to both

- **Neither initiate call is idempotent** and neither accepts an `Idempotency-Key`. A
  replayed POST creates a second request that reaches a real counterparty. Reconcile
  before retrying a timeout.
- **No webhooks.** CertifID publishes no consumer event surface, so completion is
  discovered by polling only.
- **Statuses come from `PublicRequestStatus`.** A rejected request carries a
  `RejectFeedback` object explaining why.
- **`403` means the product is not enabled** for the authenticated company — an
  entitlement problem, not an auth problem.
- Certificates are PDFs, and the `404` on certificate endpoints may be served as
  `application/pdf`.

See `conventions/certifid-conventions.yml` for the full cross-cutting semantics and
`errors/certifid-problem-types.yml` for the two error envelopes.
