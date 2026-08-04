# LendKey

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

LendKey Technologies is a New York based digital lending platform that lets credit unions and community banks originate, fund and service consumer loans without building the technology themselves. Its network-lending model covers private student loans, student loan refinancing, home improvement lending and auto lending, and its ALIRO platform runs loan participations and post-origination liquidity between institutions. LendKey has surpassed $8 billion in loan originations.

- Website — https://www.lendkey.com/
- Developer portal — https://developer.lendkey.com/default/
- Backed by: threshold-ventures

## APIs

LendKey publishes four OpenAPI 3.0 documents through a Kong Gateway fronted developer portal. Every API uses the OAuth2 client-credentials grant against a per-route `/oauth2/token` endpoint and expects `Authorization: Bearer`. Access is partner gated — credentials are issued through the portal.

| API | Base URL | Surface |
|---|---|---|
| Integration API | `https://proxy.kong.lkeyprod.com/integration/` | Leads, soft credit pull, combined monthly debt, credit attributes, scoring, application boarding, application status, transactional email |
| Treasury Management API | `https://api.lendkey.com/TreasuryMgmt` | Loan creation, disbursement create/cancel, subledger payments, capital ledger external ids |
| E-Sign API | `https://api.lendkey.com/esign` | DocuSign envelope creation, lender templates, envelope status, signer details, embedded signing links |
| Partner Integration Internal API | `https://api.lendkey.com/lo_partnerintegrationinternalapi` | Internal request/response logging for the partner integration surface |

## Artifacts

| Directory | Contents |
|---|---|
| `openapi/` | Four specs harvested verbatim from the Kong Developer Portal |
| `overlays/` | API Evangelist enrichment overlays, one per spec |
| `authentication/` | OAuth2 client-credentials profile derived from the specs |
| `scopes/` | OAuth2 schemes and token endpoints (all flows declare empty scope maps) |
| `conventions/` | Gateway behavior, auth style, error envelopes, pagination, versioning |
| `errors/` | Error catalog derived from every 4xx/5xx response, both envelope shapes |
| `lifecycle/` | Versioning, environments, deprecation and status-page findings |
| `sandbox/` | DEV / QA / Staging / Production environment matrix and credential issuance |
| `conformance/` | Standards conformance derived from the specs |
| `data-model/` | Entity graph across origination, ledger and contracts |
| `mcp/` | Candidate MCP tool surface derived from the 30 published operations |
| `skills/` | Three Agent Skills for the marquee flows |
| `asyncapi/` | Webhook surface record (inbound DocuSign receiver only) |
| `packages/` | Negative result — no first-party SDKs published |
| `well-known/` | Negative result — no `.well-known` documents on any host |
| `llms/` | `llms.txt` fetched verbatim from lendkey.com |
| `security/` | Probed TLS/HSTS/DNSSEC/CAA/SPF/DMARC posture |

## Notable gaps

- No `Idempotency-Key` header or documented idempotent-retry contract, on APIs whose writes move money.
- No consumer-subscribable events or webhooks — polling only.
- No RFC 9457 problem details, no error codes, and two different error envelopes across the estate.
- No published rate limits, deprecation policy, SLA or changelog.
- `status.lendkey.com` is provisioned on Site24x7 StatusIQ but is currently inactive/private.
- The Integration API declares no `operationId` values.
