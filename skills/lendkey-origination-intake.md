---
name: LendKey origination intake — lead, credit, score, board
description: >-
  Capture a lead, run a soft credit pull and combined monthly debt, resolve credit
  attributes, score the applicant, board the application, then poll its status using
  the LendKey Integration API.
api: openapi/lendkey-integration-openapi.yml
generated: '2026-07-19'
method: generated
source: openapi/lendkey-integration-openapi.yml
operations:
  - POST /v1/oauth2/token
  - POST /v1/leads/create
  - POST /v1/credit/soft_pull
  - POST /v1/credit/combined_monthly_debt
  - POST /v1/credit/resolve_attributes
  - POST /v1/scoring/score
  - POST /v1/onboarding/onboard
  - POST /v1/application/status
  - POST /v1/notifications/emails
  - GET /v1/notifications/emails/templates
---

# LendKey origination intake

The Integration API is LendKey's third-party integration surface for loan
origination: leads, credit, scoring, application boarding, status and transactional
email. Its own description states it "represents the integration APIs LendKey offers
for 3rd party dependencies/integrations, and is for dev/qa use only" — confirm your
production entitlement with LendKey before pointing at the production host.

**This API declares no `operationId` values.** Reference operations by method and
path, exactly as listed above.

## Environments

| Environment | Base URL |
|---|---|
| Production | `https://proxy.kong.lkeyprod.com/integration/` |
| Staging | `https://proxy.kong.lkeystaging.com/integration/` |
| QA | `https://proxy.kong.eks-1.use-1.lkeyqa.com/integration/integration-api-spring-boot-main/` |

QA is the environment LendKey directs development and testing to. **Soft credit pull
creation is disabled in Staging** — coordinate with LendKey if you need credit-pull
testing there.

## 1. Authenticate

`POST /v1/oauth2/token`. This API's token call differs from the Kong-fronted APIs:

- Header: `Authorization: Basic <base64(clientId:clientSecret)>`
- Body (form-encoded): `grant_type=client_credentials`

The token is valid for **60 minutes** here (not the 2 hours the other LendKey APIs
issue). Send `Authorization: Bearer <access_token>` on subsequent calls.

## 2. Create a lead

`POST /v1/leads/create` with `email`, `firstName`, `lastName`, `source`, `sourceKey`,
`sourceApplicationId`, `creditRiskModel`.

`sourceApplicationId` is your identifier — set it deterministically from your own
record so you can reconcile after a failed call. There is no idempotency key on this
API, so a retried lead creation may produce a duplicate.

## 3. Credit and scoring

- `POST /v1/credit/soft_pull` — soft credit pull, returning the applicant's credit
  profile. Body: `firstName`, `lastName`, `ssn`, `street`, `city`, `state`,
  `zipCode`, `creditBureauName`, `creditRiskModel`, `preamble`, `subCode`.

  **This has a real-world side effect.** A soft pull is a bureau interaction against
  a real consumer. Never retry it blind after a timeout or 500 — reconcile first.
  Never call it speculatively, in a loop, or on data you are not authorised to pull.
  Confirm you hold the applicant's permissible purpose before calling.

- `POST /v1/credit/combined_monthly_debt` — combined monthly debt for the applicant set.
- `POST /v1/credit/resolve_attributes` — resolves credit attributes used by scoring.
- `POST /v1/scoring/score` — takes an `attributes` object. The response is
  polymorphic: a pass shape (`scoringResponsePass`) or a fail shape
  (`scoringResponseFail`). **Branch on which shape came back** rather than assuming
  a single schema.

## 4. Board the application

`POST /v1/onboarding/onboard` with a `boardApplicationRequest`:

- `submissionUuid`, `applicationType`, `referenceNumber`, `preferredLanguage`
- `applicants[]` — each with `role`, `name`, `dob`, `ssn`, `driversLicense`,
  `address`, `phoneNumbers[]`, `preferredPhone`, `email`, `employer`,
  `totalAnnualIncome`, `otherIncome`, `monthlyHousingExpense`, `eligibility`,
  `relationship`, `militaryStatus`, `isMarried`, `returningApplicantId`
- `project` — the financed home-improvement project: `projectCost`, address fields,
  `housingStatus`, `currentHomeValue`, `mortgageBalance`, `monthlyHousingPayments`,
  `projectType`, `projectDescription`, `projectExpectedDuration`, `projectStatus`
- `selectedOffer` — `programId`, `fomQueryId`, `termInMonths`, `loanRate`,
  `rateIndex`, `maximumLoanAmount`
- `authorizedUser`, `consentOfDisclosures`, `contractorNetworkId`, `contractorId`,
  `requestedLoanAmount`, `merchantUseOnly`, `promoCode`

`consentOfDisclosures` records a legal consent. Only set it true when the applicant
genuinely gave that consent in your flow — never default it.

## 5. Poll status

`POST /v1/application/status` with `requester` (a `uuid`) and `referenceKeys[]`.
Returns per-application `loanKey`, `loanStatus`, `loanAmount`, and a `blockages[]`
array of human-readable strings describing what is preventing the application from
advancing (e.g. "Waiting for contractor to submit a disbursement request.").

Surface `blockages[]` to the operator — it is the actionable field.

There is no webhook or event subscription. Polling is the only mechanism.

## 6. Transactional email

- `GET /v1/notifications/emails/templates` — list available templates.
- `POST /v1/notifications/emails` — send. Returns **202 Accepted**: the send is
  queued, not confirmed delivered.

## Responses, paging and errors

Responses are `application/hal+json`. HAL `page` metadata (`number`, `size`,
`totalElements`, `totalPages`) and `_links` with `href` are defined for collection
responses.

Errors use the HAL error object:

```
{ "id": "<uuid>", "status": "BAD_REQUEST", "message": "...",
  "debugMessage": null, "timestamp": "...", "subErrors": [...] }
```

`subErrors[]` gives `object`, `field`, `rejectedValue`, `message` for each violation
— use it to build precise validation feedback. **Capture the `id` UUID** on every
failure; it is the only correlator LendKey publishes and support will ask for it.

Full catalog: `errors/lendkey-problem-types.yml`.

## Handling sensitive data

Every step here handles SSNs, driver's licence numbers, dates of birth, income, and
credit bureau data on real consumers. Do not log request or response bodies, do not
place them in traces or prompts, and do not retain them beyond the system of record.
Treat the credit operations as consequential actions requiring explicit
authorisation, not as read-only lookups.

## Cross-references

- Conventions: `conventions/lendkey-conventions.yml`
- Error catalog: `errors/lendkey-problem-types.yml`
- Environments: `sandbox/lendkey-sandbox.yml`
- Entity graph: `data-model/lendkey-data-model.yml`
