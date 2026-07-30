---
name: LendKey E-Sign contract flow
description: >-
  Assign a lender template, create a DocuSign e-signature contract envelope for a
  loan application, get an embedded signing link, poll the envelope to completion,
  and read the signer's captured form data — using the LendKey E-Sign API through
  Kong Gateway.
api: openapi/lendkey-esign-openapi.yml
generated: '2026-07-19'
method: generated
source: openapi/lendkey-esign-openapi.yml
operations:
  - getOAuth2Token
  - assignLenderTemplate
  - listTemplates
  - getTemplateByLenderAndProgram
  - createApplicationContract
  - getApplicationContracts
  - getApplicationStatuses
  - generateEmbeddedSigningLink
  - getSignerDetails
---

# LendKey E-Sign contract flow

Base URL: `https://api.lendkey.com/esign` (production; the only environment LendKey
publishes for this API). Every call goes through Kong Gateway, which validates the
token, strips the `/esign` prefix, and forwards to the backend service. You never
handle backend credentials.

## 1. Authenticate

Call `getOAuth2Token` — `POST /oauth2/token`, form-encoded:

```
grant_type=client_credentials
client_id=<from the Kong Developer Portal>
client_secret=<from the Kong Developer Portal>
```

Send `Authorization: Bearer <access_token>` on every subsequent call. The token is
valid for **7200 seconds (2 hours)**. There is no refresh grant — mint a new token
proactively rather than waiting for a 401. A 401 on the token call itself means the
credential pair was rejected; re-issue it in the portal.

Credentials are issued per environment through
<https://developer.lendkey.com/default/register>. Access is partner-gated, not
self-serve.

## 2. Make sure the lender has a template

A contract cannot be created for a lender/program pair with no assigned DocuSign
template — `createApplicationContract` returns **404 (template or application not
found)**.

- `listTemplates` — `GET /templates`, optional `lender_uuid` query filter. Returns
  all templates including soft-deleted ones.
- `getTemplateByLenderAndProgram` — `GET /templates/{lender_uuid}/program/{program_id}`
  to check a specific pairing.
- `getTemplateById` — `GET /templates/{id}` for a single template.
- `assignLenderTemplate` — `POST /templates` to attach a template to a lender.
  **This is destructive:** if the lender already has an active template it is soft
  deleted and replaced. Read the current assignment first and confirm before
  overwriting.

## 3. Create the contract envelope

`createApplicationContract` — `POST /applications` with a `NewApplicationRequest`.

The payload carries the full disclosure set alongside the borrower record:
`application_uuid`, `organization_uuid`, `program_id`, `loan_identifier`,
`loan_consumation_date`, `loan_maturity_date`, `apr`, `finance_charge`,
`amount_financed`, `total_of_payments`, `variable_rate`, `term_months`,
`payment_amount`, `interest_rate_percent`, `loan_amount`, `merchant_name`, and a
`borrower` object (with `customer_uuid`, name, email, `address`, `dob`, `ssn`,
`preferred_phone`, and FICO fields). A `coborrower` may be supplied.

Success returns **204 No Content** — there is no response body, so key your own
records on the `application_uuid` you sent.

Handle these deterministically:

- **409 Conflict** — an envelope already exists for this `application_uuid`. This is
  the closest thing the API has to an idempotent create. **Treat 409 as success on a
  retry**, not as an error: LendKey documents no `Idempotency-Key` header, so a
  blind retry after a network timeout is exactly what produces this, and creating a
  second envelope for one application is the failure you are avoiding.
- **400** — validation errors; the `errors[]` array names the failing fields.
- **404** — template or application not found; go back to step 2.

## 4. Get the borrower signing

`generateEmbeddedSigningLink` —
`GET /applications/{application_uuid}/signers/{customer_uuid}/embedded-link?return_url=<your-url>`.

The returned URL is **time-limited and single-use in practice** — generate it at the
moment you are about to render the iframe, never in advance and never cached.
`return_url` is required; the signer is redirected there whether they complete or
cancel, so do not treat a redirect as proof of signature — confirm via step 5.

## 5. Poll to completion

There is no outbound webhook you can subscribe to. LendKey's `/webhooks/docusign`
endpoint is an **inbound** receiver for DocuSign Connect, not a callback you can
register. Poll instead:

`getApplicationStatuses` — `GET /applications/{application_uuid}/statuses` returns
the envelope status history (`EnvelopeStatus[]`).

Poll with backoff. Do not poll faster than you need — no rate limits are published,
which means none are guaranteed either.

## 6. Read the signer's data

`getSignerDetails` — `GET /applications/{application_uuid}/signers/{customer_uuid}`
returns signer information plus the form-field data collected during signing.

**Only available after the envelope is completed.** Calling it earlier returns
**404 (signer not found)** — that 404 means "not yet", not "wrong id". Confirm
completion in step 5 before calling this, and treat a 404 here as a signal to keep
polling.

## Listing envelopes

`getApplicationContracts` — `GET /applications?lender_uuid=<uuid>&program_id=<id>`.
Both query parameters are **required**. The response is an unpaginated array of
`ApplicationEnvelopeSummary` — there are no paging parameters on this API, so expect
the full set and guard against unbounded growth on your side.

## Error handling

Errors use the flat Kong envelope: `isSuccess: false`, `statusCode`, `error`,
`errors[]`. There are no error codes or stable type URIs — branch on the HTTP status.
Full catalog: `errors/lendkey-problem-types.yml`.

## Handling sensitive data

Every payload in this flow carries PII and regulated financial data: SSN, date of
birth, FICO score, address, and the Truth-in-Lending disclosure figures. Do not log
request bodies, do not echo them into traces, and do not persist them outside the
system of record. The embedded signing link grants access to a signing session —
treat it as a credential.

## Cross-references

- Conventions and auth semantics: `conventions/lendkey-conventions.yml`
- Error catalog: `errors/lendkey-problem-types.yml`
- Environments and credential issuance: `sandbox/lendkey-sandbox.yml`
- Entity graph: `data-model/lendkey-data-model.yml`
