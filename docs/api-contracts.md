# API Contracts

## API principles

Expose versioned HTTPS JSON APIs under `/api/v1`. REST is the command/query interface; Socket.IO is only for subscribed real-time events. All contracts are defined in the shared `contracts` package using runtime validation. The API is tenant-aware from the authenticated principal; clients must not choose an arbitrary tenant ID.

## OpenAPI 3.1 / Swagger policy

Every public, tenant-facing, partner-facing, and webhook endpoint must be described in an OpenAPI 3.1 document generated or validated from the same runtime contract source used by the API. The specification documents authentication, authorization requirements, request/response schemas, error codes, pagination, idempotency requirements, rate-limit headers, asynchronous responses, and deprecation state. It is versioned with the API and published through an authenticated Swagger UI/documentation portal appropriate to the environment; production documentation must never expose secrets, internal topology, or sensitive example data.

Contract changes require automated compatibility checks. Backward-compatible additions may be released within a major API version; breaking changes require a new version or an explicitly approved migration path.

## Cross-cutting contract

- Authentication: `Authorization: Bearer <JWT>`; refresh/revocation behavior requires identity decision.
- Authorization: policy checks on every resource and action; `403` never reveals cross-tenant existence.
- Idempotency: all side-effecting externally repeatable endpoints accept `Idempotency-Key`.
- Correlation: accept/return `X-Request-Id`; propagate it to jobs, logs, and provider calls.
- Pagination: keyset cursor response `{ data, page: { nextCursor } }`.
- Errors: `{ error: { code, message, details?, requestId } }` with stable machine-readable codes.
- Dates: ISO-8601 UTC strings; money/counters as explicitly documented integer/decimal representations.
- Long-running work: return `202 Accepted` with `{ jobId, status, statusUrl }`; clients poll the status URL or receive an authorized Socket.IO progress event. A `202` response confirms durable acceptance, not completed provider/worker execution.

## Default API rate-limiting strategy

Apply distributed, token-bucket rate limits at the API edge and enforce stricter limits in modules/providers where required. Default limits are per authenticated tenant and principal/API key, with additional IP-based controls for unauthenticated endpoints and login abuse protection. Limits differ by operation class: read requests, ordinary writes, costly exports/imports, AI requests, campaign controls, and public webhooks. Provider-specific WhatsApp/CRM/AI throughput is enforced separately in workers and is not represented as a client API allowance.

Return `429 Too Many Requests` with `Retry-After` and standard rate-limit headers describing the applicable window/remaining allowance. Exact numeric limits are environment- and plan-configured, monitored for abuse, and documented in OpenAPI; they must not be hard-coded as undocumented client behavior. Webhook endpoints use signature verification, payload-size limits, replay protection, and provider-aware backpressure rather than a generic client limit alone.

## API versioning and deprecation

The API uses URI major versions such as `/api/v1`. A version remains backward compatible for its supported lifetime. Deprecated operations or fields are marked in OpenAPI, documented with a replacement/migration guide, and signaled with `Deprecation` and `Sunset` response headers where applicable.

The default sunset notice is at least 180 days for public/partner APIs, unless a critical security, legal, or provider-breaking change requires an accelerated retirement. Major version retirement requires an announced sunset date, usage reporting for affected tenants where feasible, migration support, and removal only after the sunset period. Undocumented behavior is not a supported contract.

## Resource surface

| Resource                    | Representative endpoints                                                                                                                                                                                                                                      |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Authentication / membership | `POST /auth/login`, `POST /auth/refresh`, `GET /me`, `GET /memberships`                                                                                                                                                                                       |
| Contacts                    | `GET/POST /contacts`, `GET/PATCH /contacts/{id}`, `POST /contacts/{id}/consents`, `POST /contacts/imports`                                                                                                                                                    |
| Media assets                | `POST /media/uploads` to request an authorized upload, `POST /media/uploads/{id}/complete` to finalize/scan, `GET /media/{id}`, `GET /media/{id}/download`; binary content transfers directly to approved object storage through short-lived authorized URLs. |
| Conversations & messages    | `GET /conversations`, `GET /conversations/{id}`, `PATCH /conversations/{id}`, `GET/POST /conversations/{id}/messages`, `POST /conversations/{id}/assignments`                                                                                                 |
| Templates                   | `GET/POST /templates`, `GET /templates/{id}`, `POST /templates/{id}/submit`, `POST /templates/sync`                                                                                                                                                           |
| Campaigns                   | `GET/POST /campaigns`, `GET/PATCH /campaigns/{id}`, `POST /campaigns/{id}/validate`, `POST /campaigns/{id}/schedule`, `POST /campaigns/{id}/pause`                                                                                                            |
| Chatbots/automation         | `GET/POST /bot-flows`, `PATCH /bot-flows/{id}`, `POST /bot-flows/{id}/publish`, `GET/POST /automation-rules`                                                                                                                                                  |
| AI                          | `POST /ai/suggestions`, `POST /ai/summaries`, `POST /ai/classifications`; all require explicit authorized context                                                                                                                                             |
| Analytics                   | `GET /analytics/inbox`, `GET /analytics/campaigns`, `GET /analytics/templates`, `POST /analytics/exports`                                                                                                                                                     |
| CRM integration             | `GET/POST /integrations/crm/connections`, `POST /integrations/crm/sync`, `GET /integrations/crm/conflicts`                                                                                                                                                    |
| Provider webhooks           | `GET/POST /webhooks/whatsapp/{provider}`; unauthenticated only after provider verification/signature validation                                                                                                                                               |
| Operational health          | `GET /health`, `GET /health/live`, `GET /health/ready`; infrastructure-only, no tenant data, and protected or network-restricted as appropriate.                                                                                                              |

## Media upload contract

Media uploads use a two-step, provider-neutral workflow. `POST /media/uploads` validates authorization, tenant quota, declared content type, size, and intended classification, then returns a media-asset ID and a short-lived upload instruction. The client uploads directly to private object storage. `POST /media/uploads/{id}/complete` requests integrity verification and asynchronous malware/content scanning; the asset cannot be attached to a message/template until its lifecycle state is `available`.

`GET /media/{id}/download` returns a short-lived, authorization-scoped download instruction rather than a durable provider or storage URL. Upload completion or scanning that cannot complete promptly returns `202 Accepted` with the standard job response. Exact maximum sizes, allowed types, and scan policy are tenant/plan and compliance configuration documented in OpenAPI.

## Health, liveness, and readiness

- `GET /health/live` is a lightweight liveness probe: it confirms the process can serve requests and does not depend on external systems.
- `GET /health/ready` is a readiness probe: it confirms the application is able to accept traffic, including essential local dependencies such as database connectivity and required configuration. It must use bounded checks and never trigger costly provider calls.
- `GET /health` provides a summarized, non-sensitive service status suitable for authorized diagnostics. Detailed dependency/queue/provider health belongs in internal observability tooling, not a public endpoint.

Probe responses must be cache-controlled and safe to expose to orchestrators. The API returns a non-success status when its stated probe condition fails, enabling safe traffic removal or restart behavior.

## Asynchronous operation contract

Long-running or fan-out operations—contact imports/exports, media scanning, campaign validation/scheduling/dispatch, CRM synchronization, template synchronization, AI processing, and analytics exports—are accepted asynchronously. On durable admission, the API returns `202 Accepted` and a job resource with a stable ID, `queued`/`running`/terminal status, timestamps, progress summary where safe, result references, and normalized failure information.

Expose `GET /jobs/{jobId}` to authorized callers, with job ownership and tenant scope enforced. Where useful, resource-specific status endpoints may redirect/reference the same job. Completion is additionally announced through authorized Socket.IO events, but clients must be able to poll and recover after disconnection. Retrying a command with the same idempotency key returns the existing accepted/completed operation rather than creating duplicate dispatches, AI requests, or synchronizations.

## WebSocket contract

Authenticate during connection, authorize room joins server-side, and scope rooms as `tenant:{tenantId}` plus narrowly authorized conversation/user rooms. Events use versioned names and payload schemas: `v1.conversation.updated`, `v1.message.created`, `v1.message.status_changed`, `v1.campaign.progress`, `v1.notification.created`. Clients must reconcile via REST after reconnect; Socket.IO is not the source of truth.

## CRM API contract expectations

The exact CRM contract must be jointly agreed. At minimum it needs OAuth/service authentication, stable external IDs, contact upsert/read, activity/message creation, ownership mapping, pagination/cursor semantics, webhook or polling support, rate limits, error codes, idempotency support, and version/deprecation policy. Until agreed, integration endpoints are internal configuration and sync-control APIs—not a claim that any specific CRM field mapping exists.
