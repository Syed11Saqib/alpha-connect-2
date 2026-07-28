# Integrations

## WhatsApp Business Platform

Use a provider adapter with operations for channel registration/configuration, template synchronization, sending, status lookup, webhook verification, and inbound-event normalization. Persist provider message/template IDs and raw webhook payload metadata separately from normalized domain events. Validate webhook authenticity, acknowledge promptly after durable storage, deduplicate provider event IDs, and process in the worker.

Outbound dispatch applies Meta/provider throughput limits per phone number and tenant, validates consent and template eligibility, records an immutable send attempt, and treats provider acceptance as distinct from delivery/read/failure events.

## External-provider resilience

CRM, AI, WhatsApp/Meta, and every future provider adapter must be called through a common resilience policy while retaining provider-specific limits and error classification. External availability must never be assumed to be transactional with Alpha Connects data.

### Timeouts

Every outbound provider call has an explicit, bounded connection timeout, request/response timeout, and where applicable an overall operation deadline. The values are configured per provider operation and environment from a centrally reviewed policy; they must be shorter than the caller's own deadline and leave time for error handling. Streaming or large-media operations use separately defined bounded deadlines. A timeout is recorded as a distinct failure class and must not hold an API request thread indefinitely.

### Retries and dead-letter handling

Retry only failures classified as transient, such as network interruption, provider `429`, and eligible `5xx` responses. Use exponential backoff with jitter, provider-aware `Retry-After` handling, a configured maximum-attempt limit, and a maximum elapsed retry window. Never retry validation, authentication/authorization, consent, malformed-request, or other permanent failures without a corrective state change.

All retried operations are idempotent through provider idempotency keys or locally persisted operation/delivery identifiers. After retry exhaustion, route the item to a durable dead-letter queue with tenant, correlation, provider, sanitized failure, attempt, and remediation metadata. Dead-letter items must be observable, access-controlled, replayable after remediation, and never silently discarded.

### Circuit breakers

Apply circuit breakers independently per provider, operation class, and where necessary tenant/channel partition. A breaker opens when configured rolling failure/timeout thresholds are crossed, rejects or defers non-essential new work quickly, and protects workers from retry storms. After a cool-down it enters a limited half-open state using controlled probes; it closes only after successful recovery criteria are met.

For WhatsApp delivery, an open breaker defers dispatches while respecting message validity/expiry constraints and alerts operators. For CRM sync, it queues/degrades synchronization without blocking inbox or messaging. For AI, it returns a controlled “assistance temporarily unavailable” outcome or uses an approved policy-governed fallback provider; it must not let downstream modules bypass the AI Assistance boundary. Breaker state changes are audited and alerted.

## CRM integration

The CRM is external and API-only. Implement a configurable mapping layer rather than direct shared database coupling.

- Maintain Alpha and CRM external identifiers with source-of-truth and conflict policies per mapped entity/field.
- Use idempotent upserts, cursor checkpoints, retry/dead-letter handling, and reconciliation reports.
- Record every sync attempt, response classification, transformation version, and conflict without storing unnecessary CRM payloads.
- Receive CRM webhooks only when signature verification, replay prevention, and an explicit event contract are available; otherwise poll with cursors.
- Never make CRM availability a synchronous requirement for sending or receiving WhatsApp messages.

## AI assistance

AI is assistive by default: suggestions, summaries, classification, and draft content require clearly defined user/action policy. Provider access goes through an adapter that enforces tenant-level enablement, prompt/data minimization, redaction where required, timeouts, rate/usage limits, and audit records. Human approval is required for outbound customer messages unless product explicitly authorizes constrained autonomous actions.

## Click-to-WhatsApp ads

Normalize Meta referral/ad metadata received with inbound conversations. Store attribution with an explicit attribution window and report it separately from causation. Required confirmation: whether campaign/ad metadata will be fetched from Meta Marketing APIs or only captured from inbound referrals.

## Observability expectations

Every provider request, webhook, retry, circuit-breaker decision, and synchronization job carries a correlation ID and trace context across API, worker, queue, and adapter boundaries. Emit structured logs with provider, operation, tenant-safe identifiers, request/correlation IDs, attempt number, outcome, error class, and sanitized latency metadata; never log credentials, authorization headers, raw sensitive payloads, or unrestricted message/CRM content.

Metrics and dashboards must provide, at minimum:

- Provider request count, success/failure rate, timeout rate, and error classification by provider and operation.
- Latency distributions (including p50, p95, and p99) for outbound calls and end-to-end asynchronous jobs.
- Retry count, retry exhaustion, queue age/depth, dead-letter volume, and replay outcome.
- Circuit-breaker state, open duration, rejected/deferred work, and recovery probes.
- WhatsApp webhook ingestion/processing lag and delivery-status lag; CRM sync lag/conflicts; AI usage, rejection, and safety-policy outcomes.

Distributed traces must connect an inbound webhook or client command to its outbox event, worker execution, external call, retry sequence, and resulting domain update where sampling/privacy policies permit. Alerts are based on sustained error/timeout rates, backlog age, dead-letter growth, open breakers, and provider latency regressions, with runbooks identifying safe remediation and replay procedures.

## Secrets and configuration

Credentials belong in a managed secret store in production, referenced by configuration—not committed to source or stored plaintext in PostgreSQL. Tenant-owned integration credentials require encryption, access audit, rotation, and revocation procedures.
