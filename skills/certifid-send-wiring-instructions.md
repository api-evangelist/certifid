---
name: certifid-send-wiring-instructions
description: Securely send a title or settlement company's wiring instructions to a buyer through CertifID, poll the request to completion, and retrieve the PDF certificate that evidences the protected transfer.
generated: '2026-08-09'
method: generated
source: openapi/certifid-v2-apis-openapi.json
api: CertifID V2 APIs
base_url: https://api.certifid.com
operations:
  - PostWiringInstructionsSend
  - GetWiringInstructionsSendByRequestId
  - GetWiringInstructionsSendCertificateByRequestId
  - GetBankByAbaRoutingNumber
  - GetBankWiringDetailsByAbaRoutingNumber
---

# Send wiring instructions to a buyer

This is CertifID's flagship flow. A title or settlement agent sends its own wiring
instructions to a home buyer over a verified channel so the buyer cannot be redirected
to a fraudster's account. A completed request yields a PDF certificate that evidences
the transfer was protected.

> **operationId note.** CertifID's published specification gives an operationId to
> exactly one of its 57 operations. The names used here come from
> `overlays/certifid-certifid-v2-apis-overlay.yaml`, which assigns a deterministic id to
> each operation. Every path and method below is verbatim from the live spec.

## Before you start

- **Auth.** Every call needs an OAuth 2.0 bearer token from
  `https://auth.certifid.com/oauth/token` with audience `https://api.certifid.com`.
  See `authentication/certifid-authentication.yml`.
- **Scopes.** This flow requires `CertifIDUser` plus `RequestProduct`, and the
  per-request reads use a `ResourceAccessToken|read:requests|requestId|SendRequest`
  scope. See `scopes/certifid-scopes.yml`.
- **Entitlement.** A `403` means the authenticated account's company does not have this
  product enabled — it is not a token problem. Do not retry; escalate.

## Step 1 — optionally validate the receiving bank

`GET /api/v1/Bank/{abaRoutingNumber}` (**GetBankByAbaRoutingNumber**) resolves an
ABA/routing number to a correspondent bank's details, and
`GET /api/v1/Bank/{abaRoutingNumber}/WiringDetails`
(**GetBankWiringDetailsByAbaRoutingNumber**) returns that bank's wiring details.

Use these to catch a transposed routing number before it reaches a buyer.

## Step 2 — initiate the send request

`POST /api/v1/WiringInstructions/Send` (**PostWiringInstructionsSend**) — "Initiate a
new request to send wiring instructions to a buyer."

The `400` response on this operation documents the validation rules explicitly:

- `Recipient.FirstName`, `Recipient.LastName` and `RequestDetails.DeliveryMethod` are
  required.
- Email must be a valid format and phone number must be **10 digits**.
- The routing number, where supplied, must be valid.
- Error code `119` is `Invalid Phone Number` — the only numeric CertifID error code
  published anywhere in the specification.

**This POST is not idempotent and accepts no `Idempotency-Key` header.** Nothing in the
contract deduplicates a replay. If the call times out, do **not** blind-retry — a second
call creates a second request to a real buyer. Instead, reconcile first (see below), and
carry your own `externalReferenceId` where the model allows it so you can find your
record again.

## Step 3 — poll for completion

`GET /api/v1/WiringInstructions/Send/{requestId}`
(**GetWiringInstructionsSendByRequestId**) — "Retrieve the latest state of an initiated
send request."

Read `status` against the `PublicRequestStatus` enum. There is **no webhook and no event
stream** for this flow — CertifID publishes no consumer-facing event surface, so polling
is the only option. Back off between polls; no rate limit is documented and no `429` is
declared, so treat the ceiling as unknown and stay conservative.

## Step 4 — retrieve the certificate

`GET /api/v1/WiringInstructions/Send/{requestId}/Certificate`
(**GetWiringInstructionsSendCertificateByRequestId**) — "Get the confirmation PDF
certificate associated with a successfully completed send request."

The response is a **PDF**, not JSON. Note that the `404` on the certificate endpoints is
declared with an `application/pdf` content type, so a not-found may not parse as JSON —
branch on the status code, not the body.

## Error handling

| Status | Meaning | Action |
|---|---|---|
| `400` | Validation failure | Fix the named field. Read `errors[].code` / `.message` / `.details`. |
| `401` | Token missing/invalid/expired | Re-mint the token. No body is documented. |
| `403` | Product not enabled for the company | Escalate — retrying will not help. |
| `404` | Request id not found or not visible to the caller | Verify the id belongs to your company. |
| `500` | Unexpected error | Retry with backoff; no correlation id is returned, so capture the timestamp for support. |

Two error envelopes coexist — ASP.NET `ProblemDetails` and a CertifID `Response`
envelope with an integer `errors[].code`. Handle both. See
`errors/certifid-problem-types.yml`.
