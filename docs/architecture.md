# Architecture

## Purpose

Alpha Connects is a production-ready SaaS for operating the WhatsApp Business Platform (WABP). It provides multi-tenant messaging, automation, campaign, AI, analytics, and CRM-integration capabilities. The CRM is owned by another team and is treated exclusively as an external API dependency.

## Architectural approach

Use a **modular monolith**: one deployable backend with strict module boundaries, independently owned data access, and explicit internal contracts. Each module must be removable into a service later without requiring a cross-cutting rewrite.

The monorepo contains a Next.js 15 web application and NestJS API/worker applications. PostgreSQL is the system of record; Prisma is the database access layer. Socket.IO supplies authenticated real-time updates. Docker provides reproducible local and deployment environments.

## Logical topology

```text
Browser -> Next.js web application -> NestJS API
                                      |-> PostgreSQL
                                      |-> Socket.IO gateway
                                      |-> background job runtime
                                      |-> WhatsApp Business Platform / Meta APIs
                                      |-> CRM APIs
                                      |-> approved AI provider(s)
```

All third-party callbacks enter through dedicated webhook endpoints, are signature-verified, durably recorded, and processed asynchronously. User-facing API requests should not wait for provider delivery, campaign fan-out, AI completion, or CRM retries.

## C4-style system context and containers

```text
                                  +-------------------+
                                  | Platform operators|
                                  +---------+---------+
                                            |
                                            v
+----------------+       HTTPS / Socket.IO +-----------------------------+
| Tenant users   | ----------------------> | Alpha Connects Web (Next.js) |
+----------------+                         +--------------+--------------+
                                                       |
                                                       | HTTPS / JWT
                                                       v
                              +------------------------+------------------------+
                              | Alpha Connects API (NestJS modular monolith)    |
                              |-------------------------------------------------|
                              | Identity & RBAC | Contacts | Inbox | Templates  |
                              | Campaigns | Bots | Notifications | AI | Metrics |
                              | CRM Integration | Webhooks/Outbox | Socket.IO   |
                              +---------+---------------+--------------+-------+
                                        |               |              |
                         transactions  |               | events/jobs  | realtime
                                        v               v              v
                              +---------+----+   +------+-------+  +---+--------+
                              | PostgreSQL  |   | Worker runtime|  | Socket.IO  |
                              | operational |   | queues/retries|  | clients    |
                              | data/outbox |   +------+-------+  +------------+
                              +--------------+          |
                                                   provider calls
                                                     +-----+-----+----------------+
                                                     |           |                |
                                                     v           v                v
                                            +------------+ +-----------+ +---------------+
                                            | Meta/WABP  | | CRM APIs  | | AI providers  |
                                            +-----+------+ +-----------+ +---------------+
                                                  |
                                          verified webhooks
                                                  v
                                     API webhook ingress / inbox events

                   +-------------------+
                   | Object storage    |
                   | media & exports   |
                   +-------------------+
```

The diagram separates browser users, application containers, persistent operational data, asynchronous processing, and external systems. The worker may initially share the same codebase and deployment image as the API but runs as a separate scalable process.

## Core principles

- **Tenant isolation first:** every tenant-owned record carries `tenant_id`; authorization and query scoping are mandatory.
- **Provider boundaries:** Meta, CRM, and AI clients are adapters behind module interfaces. No provider SDK types may leak into domain modules.
- **Asynchronous reliability:** use a durable job mechanism and transactional outbox/inbox patterns for side effects and webhook ingestion.
- **Security by default:** least-privilege RBAC, encrypted secrets, JWT rotation/revocation strategy, audit trails, webhook verification, rate limits, and privacy controls.
- **Observable operations:** structured logs, correlation IDs, metrics, traces, health/readiness checks, alertable dead-letter queues, and audit events.
- **Evolution without coupling:** modules communicate through application interfaces and domain events, never by importing another module's repository or database tables.

## Deployable responsibilities

| Component                                      | Responsibility                                                                                                         |
| ---------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Web                                            | Tenant UI, authenticated server-side calls/BFF concerns, static assets. No business rules duplicated from the API.     |
| API                                            | REST API, authentication/authorization, commands and queries, provider webhook acceptance, Socket.IO fan-out.          |
| Worker                                         | Scheduled and queued work: campaign delivery, retries, syncs, notification evaluation, analytics projections, AI jobs. |
| PostgreSQL                                     | Transactional source of truth, outbox/inbox, audit data, operational aggregates.                                       |
| Object storage (decision required)             | Large exports, template media, and retained attachments where applicable.                                              |
| Redis/queue infrastructure (decision required) | Distributed queue, rate limiting, cache/presence, Socket.IO scaling adapter.                                           |

## Request and event flows

1. The web client sends an authenticated REST command to the API.
2. The API validates input, authorizes the tenant/principal, executes a module use case, and commits a transaction.
3. The transaction writes domain changes and outbox events atomically.
4. The worker publishes/consumes events and calls external systems with idempotency keys and retry policies.
5. State changes are projected to Socket.IO rooms scoped to the tenant and permissions.

For inbound WhatsApp events, acknowledge Meta promptly after validation and durable inbox storage; process the event asynchronously and idempotently.

## Data and tenancy

PostgreSQL uses a shared-database/shared-schema tenancy model with mandatory tenant predicates. Prisma access must be wrapped so tenant-scoped repositories cannot execute unscoped tenant queries. Composite unique indexes include `tenant_id` where identifiers are tenant-local. Consider PostgreSQL row-level security as defense in depth after validating Prisma and migration operational implications.

## Multi-tenancy strategy

**Recommendation: shared PostgreSQL database and schema with a mandatory tenant discriminator, enforced in application code and optionally reinforced with PostgreSQL row-level security (RLS).** This is the best initial architecture for a SaaS WhatsApp platform because it enables efficient operations, aggregate infrastructure cost, simple migrations, pooled connections, and tenant-independent horizontal scale while preserving a clear path to dedicated tenancy for enterprise customers.

Every tenant-owned entity carries an immutable `tenant_id`. The authenticated membership supplies the trusted tenant context; `tenant_id` is never accepted as a freely selectable client field. Repositories receive tenant context explicitly, all tenant-local unique indexes include `tenant_id`, and authorization checks occur before object access. Background jobs and events carry a signed/validated tenant context and must use the same scoped repository path.

Defense in depth should include:

- Tenant-scoped query helpers/repositories as the normal Prisma access path, with code review and automated tests that reject unscoped data access.
- RLS policies for high-risk tenant tables after operational validation, with the database connection setting tenant context for every transaction.
- Separate encryption keys or key-encryption-key scopes per tenant where contractual requirements warrant it.
- Export, audit, backup, and observability processes that preserve tenant boundaries and redact sensitive content.

For future enterprise isolation, keep the tenant resolver and repository interfaces independent from physical database placement. A routing layer can later direct selected tenants to a dedicated database without changing module business logic. Dedicated databases should be an explicit commercial/compliance tier, not the default.

## Storage architecture for media and documents

PostgreSQL stores media/document metadata, ownership, content type, size, checksum, lifecycle state, and provider references; it must not store large binary payloads as ordinary application rows. Use private, S3-compatible object storage (or a cloud-managed equivalent) for inbound/outbound attachments, template media, generated exports, and approved retained documents.

Objects are partitioned by environment and tenant prefix, for example `tenants/{tenant-id}/{classification}/{object-id}`. The API issues short-lived, least-privilege upload/download URLs only after authorization. It validates declared type/size, runs asynchronous malware scanning before making uploads available, derives safe previews where required, and records immutable audit events for access and deletion. Object encryption at rest uses the storage provider's encryption plus managed keys where needed; transport is TLS only.

Retention rules must differentiate operational attachments, temporary uploads, exports, and provider-retrievable media. Deletion is a lifecycle workflow: mark metadata unavailable, prevent new access, delete derived copies, and expire the object according to the approved retention/legal-hold policy. Exact storage provider, maximum file limits, scanning service, regional residency, and retention schedules remain decisions requiring confirmation.

## Internal module communication

Modules communicate through versioned domain events and narrow application interfaces rather than direct repository/table access. A module commits its own state change and an `outbox_events` record in one database transaction. An internal event dispatcher delivers events to subscribed modules; workers may process asynchronous subscribers. The event envelope includes `event_id`, `event_type`, `schema_version`, `occurred_at`, `tenant_id`, `correlation_id`, causation ID, and minimal business payload.

Use synchronous interfaces only where a request cannot complete without a bounded, in-process business decision—for example, checking contact consent before a send command. These interfaces expose stable application-level contracts, never infrastructure repositories. Event consumers must be idempotent, tolerate duplicate/out-of-order delivery, record processing status, and support replay. This approach allows the dispatcher to be replaced by an external broker during a future module extraction without changing publishers or consumers' domain contracts.

## Realtime inbound message flow

```text
Meta webhook
  -> Webhook controller verifies challenge/signature and normalizes request metadata
  -> Transactionally persists immutable webhook inbox event (unique provider event ID)
  -> Returns fast acknowledgement to Meta
  -> Worker claims inbox event and translates it into Inbox/Contact domain changes
  -> Transaction commits message/conversation updates plus outbox event
  -> Realtime projection subscriber authorizes target tenant/user rooms
  -> Socket.IO emits versioned message/conversation event
  -> Frontend updates its local view and reconciles through REST after reconnect
```

The webhook HTTP path does not execute slow automation, CRM synchronization, AI work, media retrieval, or Socket.IO delivery directly. Socket.IO is a low-latency notification channel, not the source of truth; the database remains authoritative. If a client is offline, it fetches the current conversation state using cursor-based REST endpoints.

## RBAC architecture

Authorization is tenant-scoped and policy-based. A user may hold different memberships and roles in different tenants. Roles are collections of atomic permissions in the form `resource.action`, and policies additionally evaluate scope (tenant, assigned conversations, ownership, channel access, or sensitive-data access). API and WebSocket authorization use the same policy engine.

| Role                | Typical permissions                                                                                                                |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| Tenant Owner        | Full tenant administration, billing/integration administration, role management, destructive settings.                             |
| Tenant Admin        | User/channel/template/campaign/automation management; operational reporting; no ownership transfer unless granted.                 |
| Inbox Manager       | View and manage all permitted conversations, assignments, tags, and team queue configuration.                                      |
| Agent               | View/reply to assigned or explicitly permitted conversations; contact updates within scope; no channel/integration administration. |
| Campaign Manager    | Create, validate, schedule, pause, and view campaigns/templates within approved channels; no role management.                      |
| Analyst             | Read-only, permission-filtered analytics and export access.                                                                        |
| Integration Manager | Configure CRM/provider connections and inspect sync failures; credentials remain masked and audited.                               |

Representative atomic permissions include `conversation.read`, `conversation.reply`, `conversation.assign`, `contact.write`, `template.submit`, `campaign.schedule`, `automation.publish`, `analytics.export`, `integration.manage`, `member.manage`, and `audit.read`. Roles are configurable only within a controlled permission catalog; role changes, exports, credential actions, and access to sensitive content are audited. Exact default role matrix and whether customers need custom roles require product confirmation.

## AI abstraction layer

AI business features depend on an internal `AI Gateway` application port, not a specific provider SDK. The port accepts a provider-neutral request containing capability (suggest reply, summarize, classify, extract), authorized tenant/context references, model policy, output schema, and safety requirements; it returns a normalized result with usage, provenance, confidence where meaningful, and failure classification.

Provider adapters implement this port for approved vendors. The gateway chooses an enabled provider/model by tenant policy and capability, applies prompt templates, redaction/data-minimization, structured-output validation, timeout/retry limits, content-safety controls, and usage/cost limits. It records auditable request metadata without retaining sensitive prompt content beyond the approved policy. Application modules consume stable AI results, so adding, replacing, or failing over providers does not change Inbox, Chatbot, Campaign, or CRM business logic.

## Broadcast processing architecture

Campaign creation validates permissions, template/channel eligibility, consent/suppression policies, and an immutable audience snapshot. Scheduling creates a campaign execution and durable dispatch work records; it does not synchronously send messages from the API request.

```text
Campaign command -> campaign/audience transaction -> outbox event
  -> scheduler/worker materializes recipient dispatches
  -> queue partitions by tenant + WhatsApp phone number
  -> rate-limited workers validate final eligibility and send through channel adapter
  -> provider acceptance/status webhooks update dispatch state
  -> event projections update campaign metrics and Socket.IO progress
```

Workers use bounded concurrency, per-channel/provider rate limits, idempotency keys, delayed retries with exponential backoff and jitter, retry classification, and dead-letter queues. Dispatch records preserve each recipient's resolved template/version and render inputs needed for auditability without retaining unnecessary sensitive data. Pause/cancel operations stop future claims safely; already accepted provider sends are reconciled through delivery status events. The campaign is complete only after all recipient dispatches reach a terminal or explicitly recoverable state.

## Failure strategy

- Idempotent webhook handling keyed by provider event ID.
- Outbox delivery with exponential backoff, jitter, max-attempt policy, and dead-letter visibility.
- Provider-specific rate limits enforced by the dispatch layer.
- Circuit breakers and timeouts around CRM, Meta, and AI calls.
- Compensating status transitions, never distributed database transactions with external systems.
- Reconciliation jobs for provider delivery status and CRM synchronization.

## Decisions requiring confirmation

- Target WhatsApp integration: Cloud API directly, a BSP, or both.
- Queue/cache technology and managed hosting choice.
- Identity provider and SSO requirements beyond JWT.
- Data residency, retention, deletion, and legal/compliance requirements.
- AI provider, permitted data classes, and human-approval policy.
