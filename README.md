# LendKey

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
