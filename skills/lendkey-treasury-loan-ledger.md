---
name: LendKey Treasury loan ledger operations
description: >-
  Create loans, record and cancel disbursements, post payments to the subledger, and
  update capital ledger external ids using the LendKey Treasury Management API
  through Kong Gateway — including correct handling of 207 Multi-Status partial
  failures.
api: openapi/lendkey-treasury-management-openapi.yml
generated: '2026-07-19'
method: generated
source: openapi/lendkey-treasury-management-openapi.yml
operations:
  - getOAuth2Token
  - createLoan
  - createDisbursement
  - cancelDisbursement
  - createPayment
  - updateExternalId
---

# LendKey Treasury loan ledger operations

This API manages loan inventory, disbursements, payments and treasury entries. It is
fronted by Kong Gateway, which validates your token, strips the environment route
prefix, injects the backend OutSystems headers (`X-Auth-Key`, `X-Auth-AppId`) and
forwards the call. You never manage OutSystems credentials.

## Environments

| Environment | Base URL |
|---|---|
| DEV | `https://api.lkeyqa.com/TreasuryMgmtDev` |
| QA | `https://api.lkeyqa.com/TreasuryMgmtQa` |
| Staging | `https://api.lkeystaging.com/TreasuryMgmt` |
| Production | `https://api.lendkey.com/TreasuryMgmt` |

Route names are case-insensitive. Each environment issues its own credentials.
**Always confirm which base URL you are pointed at before any write** — the payloads
are identical across environments, so a misconfigured base URL silently posts real
ledger entries.

## 1. Authenticate

`getOAuth2Token` — `POST <base>/oauth2/token`, form-encoded with
`grant_type=client_credentials`, `client_id`, `client_secret`. Send
`Authorization: Bearer <access_token>` on every call. Token lifetime **7200 seconds**;
no refresh grant, so re-mint proactively.

## 2. The response envelope — read it, do not trust the HTTP status

Every write returns `isSuccess`, `statusCode`, and `responseDetails[]`. Writes can
return **HTTP 207 Multi-Status**: the batch was accepted but individual items may
have failed, with per-item messages in `responseDetails[].errors`.

**A 2xx does not mean every item succeeded.** Always:

1. Check `isSuccess`.
2. Walk `responseDetails[]` and inspect each `errors[]` array.
3. Retry only the items that failed, reusing the same caller-supplied identifier.

## 3. Create a loan

`createLoan` — `POST /v1/createLoan`. The body is an **array** of `CreateLoanRequest`
objects (batch create).

Required per item: `name`, `externalID`, `applicationID`, `maxPrincipal`,
`productName`, `term` (months). Optional: `startDate`, `endDate`, and nested
`disbursement[]`, `loanPerson[]` (`personId` + `role`, e.g. Borrower) and
`loanParty[]` (`partyId` + `role`, e.g. Lender).

The response returns a generated loan `uuid` per item alongside the `externalId` you
supplied. **Persist that uuid** — `createDisbursement` needs it.

`applicationID` is how the ledger joins back to origination — carry the application
identifier from the Integration API through unchanged.

## 4. Disbursements

- `createDisbursement` — `POST /v1/createDisbursement`. Requires `loanUUID` (the uuid
  from step 3), `amount`, `externalID`; optional `startDate`. Returns a disbursement
  `uuid`.
- `cancelDisbursement` — `POST /v1/cancelDisbursement` reverses one. All seven fields
  are required: `subledger_Category` (must be the literal
  `"Disbursement Cancellation"`), `external_id`, `account_id`, `entity_type`,
  `amount`, `disbursement_uuid`, `created_by`. Note this operation uses
  **snake_case** while `createDisbursement` uses camelCase — the casing is not
  consistent across operations, so do not reuse a serializer blindly.

## 5. Payments

`createPayment` — `POST /v1/createPayment`. Required: `subledgerCategory` (literal
`"Payment"`), `isSubledgerReversal`, `subledgerItemCode`, `paymentId`,
`payorEntityType`, `amount`, `loanExternalId`. Optional `purpose`.

`subledgerItemCode` is an enum: `INTEREST`, `PRINCIPAL`, `Late Fee`, `NSF Fee`,
`Other`. Note the inconsistent casing in the enum values — send them verbatim.

Set `isSubledgerReversal: true` to reverse a previously posted payment.

Payments reference the loan by `loanExternalId` (your external id), **not** by the
loan uuid — unlike disbursements, which use `loanUUID`.

## 6. Ledger maintenance

`updateExternalId` — `PATCH /v1/updateExternalId` with `capitalLedgerTargetId` and
the new `externalId`. Updates the external identifier on a Capital Ledger Target row.

## Idempotency — there is none, so build it yourself

LendKey documents **no** `Idempotency-Key` header and no idempotent-retry contract on
this API. Every write is a money-moving or ledger-affecting operation, and a blind
retry after a timeout risks a duplicate loan, disbursement or payment.

Protect yourself:

- Always supply a deterministic caller-side identifier — `externalID` on loans and
  disbursements, `paymentId` on payments — derived from your own record, never a
  random value per attempt.
- On a timeout or 5xx, **reconcile before retrying**. Do not retry a payment or
  disbursement blind.
- Treat 207 as partial failure and retry only the failed items, reusing the same
  identifiers.

## Errors

Flat envelope: `isSuccess: false`, `statusCode`, `error`, `errors[]` (field-level
messages). 400 validation, 401 expired/invalid token, 500 server error. No error
codes or type URIs. Full catalog: `errors/lendkey-problem-types.yml`.

## Cross-references

- Conventions: `conventions/lendkey-conventions.yml`
- Environments and credentials: `sandbox/lendkey-sandbox.yml`
- Entity graph: `data-model/lendkey-data-model.yml`
