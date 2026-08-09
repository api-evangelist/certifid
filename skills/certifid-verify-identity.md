---
name: certifid-verify-identity
description: Initiate a CertifID identity verification (IDV) request for a transaction participant, poll it to a terminal state, and retrieve the PDF certificate evidencing the verification.
generated: '2026-08-09'
method: generated
source: openapi/certifid-v2-apis-openapi.json
api: CertifID V2 APIs
base_url: https://api.certifid.com
operations:
  - CreateIdentityVerification
  - GetIdentityIdentityVerificationByRequestId
  - GetIdentityGetIdentityRequestCertificateCertificateByRequestId
---

# Verify a participant's identity

CertifID's IDV flow confirms that a buyer, seller or other party to a closing is who they
claim to be. It maps to the **Match / IDV** component on the CertifID status page.

## Auth and scopes

OAuth 2.0 bearer token, audience `https://api.certifid.com`. Requires `CertifIDUser`;
per-request reads use `ResourceAccessToken|read:requests|requestId|IdentityRequest`.

A `403` on this flow is documented explicitly as "the authenticated account's company
does not have identity verification enabled" — an entitlement problem. Do not retry.

## Step 1 — initiate

`POST /api/v1/identity/IdentityVerification` — **CreateIdentityVerification**.

This is **the only operation in the entire CertifID specification that carries a
publisher-assigned `operationId`**; every other name in this repo's skills comes from
our overlay.

Summary: "Initiates a new identity verification request for a recipient."

The `400` documents the same validation family as the wiring flows — required recipient
name fields, valid email format, 10-digit phone — and surfaces error code `119`
(`Invalid Phone Number`).

**Not idempotent, no `Idempotency-Key`.** A replay sends a second verification request to
a real person. Reconcile before retrying.

## Step 2 — poll

`GET /api/v1/identity/IdentityVerification/{requestId}` —
**GetIdentityIdentityVerificationByRequestId**: "Retrieves the latest state of an
initiated IDV request."

Read against the `PublicIdentityRequestStatus` enum; details arrive in a
`VerificationDetails` object. There is no webhook you can subscribe to, so poll with
backoff.

## Step 3 — certificate

`GET /api/v1/identity/GetIdentityRequestCertificate/{requestId}/Certificate` —
**GetIdentityGetIdentityRequestCertificateCertificateByRequestId**. Returns the PDF
certificate for a successfully completed IDV.

## A note on the "webhook" endpoint

The specification contains
`POST /api/v1/identity/IdVerificationResultUrlWebhook/{requestId}`, summarised as
"Webhook that is called with id verification results - results_url".

**This is not an event you can subscribe to.** It is an inbound receiver that CertifID
exposes so its own identity-verification vendor can post results back. Integrators do not
call it and cannot register a URL against it. CertifID publishes no consumer-facing
webhook catalog and no AsyncAPI — polling is the only completion signal available.

## Handling personal data

This flow moves identity documents and personal information about real people. Retrieve
only the fields you need, do not persist certificate PDFs longer than your own retention
policy allows, and remember that no correlation id is returned on errors — capture
timestamps rather than echoing payloads into logs.
