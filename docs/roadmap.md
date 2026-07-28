# Development Roadmap

## Phase 0 — Discovery and architecture sign-off

Confirm WhatsApp provider model, CRM API specification, tenant/role model, compliance and retention obligations, notification channels, AI governance, scale targets, regions, SLOs, and deployment environment. Approve the documents in this directory before implementation.

### Sprint milestones

- **Sprint 0.1 — Product and integration discovery:** Confirm user journeys, WhatsApp provider/BSP model, CRM API ownership, integration authentication, and operational constraints.
- **Sprint 0.2 — Security and data governance:** Agree tenant model, roles, data classification, retention/deletion, residency, audit, and AI governance requirements.
- **Sprint 0.3 — Architecture approval:** Review and approve architecture, module boundaries, database design, API contracts, integration resilience, roadmap, and acceptance criteria.

### Definition of Done

- WhatsApp provider and CRM integration assumptions are documented and owned by named stakeholders.
- Architecture decisions, open decisions, risks, dependencies, scale/SLO targets, and non-functional requirements are recorded and approved.
- The document set in `docs` is reviewed with engineering, product, security, and integration stakeholders.
- Implementation scope, release sequencing, and approval authority are explicit.

## Phase 1 — Platform foundations

Establish monorepo governance, CI quality gates, container strategy, configuration/secrets approach, database migration policy, observability baseline, tenant isolation controls, authentication/RBAC, audit logging, and integration reliability primitives (inbox/outbox/jobs). Exit criterion: secure tenant-aware foundation validated by automated tests and operational checks.

### Sprint milestones

- **Sprint 1.1 — Engineering foundation:** Establish monorepo governance, branch/PR policy, CI quality gates, container/deployment conventions, and environment configuration strategy.
- **Sprint 1.2 — Secure platform core:** Deliver tenant context, authentication/RBAC design, audit logging, secrets handling, database migration policy, and access-control verification.
- **Sprint 1.3 — Operational foundation:** Establish outbox/inbox/job primitives, baseline health checks, structured logging, metrics/tracing conventions, and operational dashboards/runbooks.

### Definition of Done

- CI enforces type, lint, test, build, dependency/security, and migration validation gates.
- Tenant isolation, authorization, audit, secrets, and configuration controls are covered by automated verification and security review.
- Database changes follow an approved, reversible expand-migrate-contract policy.
- Health/readiness/liveness, correlation IDs, structured logs, basic metrics/traces, and failure visibility are operationally verified.
- Inbox/outbox/job reliability primitives have documented retry, idempotency, and dead-letter behavior.

## Phase 2 — WhatsApp channel and shared inbox

Deliver provider onboarding/configuration, verified webhook intake, normalized inbound/outbound messaging, message status tracking, contact identity/consent basics, conversations, assignments, notes, and authorized real-time inbox updates. Exit criterion: operators can reliably handle a production conversation lifecycle with traceability.

### Sprint milestones

- **Sprint 2.1 — Provider and webhook foundation:** Establish WhatsApp channel onboarding, credentials/configuration lifecycle, signature verification, durable webhook ingestion, and normalized provider events.
- **Sprint 2.2 — Conversation operations:** Deliver contact identity/consent basics, conversations, messages, status updates, assignments, notes, and authorization scopes.
- **Sprint 2.3 — Realtime operator experience:** Deliver authorized Socket.IO updates, agent presence, inbox query performance, reconciliation behavior, and operator auditability.

### Definition of Done

- Inbound webhook events are verified, durable, idempotent, replayable, and processed asynchronously.
- Operators can send/receive, assign, update, and audit a full permitted conversation lifecycle.
- Consent and template/session eligibility are enforced before outbound send attempts.
- Message delivery/read/failure statuses reconcile to provider events and remain traceable.
- Realtime updates are tenant/permission scoped and recover correctly after client reconnection.

## Phase 3 — Templates, campaigns, and notifications

Deliver template lifecycle and synchronization, compliant campaign audience snapshots and scheduling, rate-limited dispatch, suppression/opt-out controls, delivery reporting, and event-based/scheduled notifications. Exit criterion: large sends are resumable, observable, idempotent, and policy compliant.

### Sprint milestones

- **Sprint 3.1 — Template operations:** Deliver template creation/versioning, provider synchronization, approval/status handling, media references, and audit history.
- **Sprint 3.2 — Campaign execution:** Deliver immutable audiences, preflight validation, scheduling, queue-based rate-limited dispatch, pause/cancel, and recovery behavior.
- **Sprint 3.3 — Notifications and reporting:** Deliver suppression/opt-out controls, notification rules, delivery projections, campaign progress, and operational reporting.

### Definition of Done

- Template lifecycle and provider statuses are synchronized, versioned, and auditable.
- Campaign recipients are reproducible from immutable audience snapshots and comply with consent/suppression policy.
- Dispatches are idempotent, rate-limited, resumable, observable, and protected by retry/dead-letter controls.
- Pause/cancel behavior prevents future eligible work while reconciling already accepted sends.
- Delivery reporting reconciles with message status events and exposes actionable failures.

## Phase 4 — Automation and CRM integration

Deliver versioned chatbot flows with handoff, automation rules, CRM connection/mapping/sync/reconciliation, and conflict handling based on the agreed CRM contract. Exit criterion: integration failures do not interrupt messaging and are actionable by operators.

### Sprint milestones

- **Sprint 4.1 — Automation foundation:** Deliver versioned chatbot flow definitions, publish controls, trigger evaluation, execution sessions, handoff, and automation-rule governance.
- **Sprint 4.2 — CRM connection and mapping:** Deliver approved CRM authentication, connection lifecycle, external-ID mapping, field ownership/source-of-truth rules, and sync configuration.
- **Sprint 4.3 — Sync resilience and operations:** Deliver asynchronous sync, cursors/webhooks as agreed, conflict management, reconciliation, observability, retries, and operator remediation.

### Definition of Done

- Chatbot flows are versioned, permission controlled, testable, publishable, and can safely hand off to an agent.
- CRM synchronization uses the agreed API contract, idempotency, mapping versioning, and explicit field-conflict policies.
- CRM outages/degradation do not block WhatsApp inbound or outbound messaging.
- Sync failures, conflicts, retries, and dead-letter items are observable and remediable by authorized operators.
- Reconciliation demonstrates consistency against the agreed CRM source-of-truth model.

## Phase 5 — AI, ads, and analytics

Deliver governed AI assist features, Click-to-WhatsApp attribution, operational and campaign analytics, exports, usage controls, and data-quality monitoring. Exit criterion: all derived reporting is reconcilable to source events and AI use is auditable.

### Sprint milestones

- **Sprint 5.1 — Governed AI assistance:** Deliver provider-neutral AI gateway capabilities, tenant enablement, data/prompt controls, human approval, usage limits, and audit outcomes.
- **Sprint 5.2 — Ads attribution:** Deliver normalized Click-to-WhatsApp referral capture, attribution definitions, and reporting boundaries.
- **Sprint 5.3 — Analytics and exports:** Deliver event projections, operational/campaign reporting, asynchronous exports, data-quality checks, and reconciliation tooling.

### Definition of Done

- AI features use only the AI Assistance boundary, apply approved safety/privacy policy, and retain auditable usage/provenance records.
- No AI-generated outbound customer content is sent without the approved human/autonomous policy.
- Ad attribution is explicitly labeled as attribution and reconciles to captured provider referral data.
- Analytics definitions, source events, freshness expectations, and export access controls are documented.
- Reported metrics and exports reconcile to operational source events within agreed tolerances.

## Phase 6 — Production hardening

Perform load/soak, security, disaster-recovery, backup-restore, webhook replay, provider outage, CRM outage, and tenant-isolation tests. Define runbooks, on-call alerts, SLO dashboards, penetration-test remediation, and rollout/rollback procedures. Exit criterion: agreed readiness review passed.

### Sprint milestones

- **Sprint 6.1 — Reliability validation:** Execute load/soak, queue/backlog, webhook replay, provider outage, CRM outage, and tenant-isolation scenarios against agreed targets.
- **Sprint 6.2 — Security and recovery:** Complete security review/penetration-test remediation, backup-restore tests, disaster-recovery exercises, and retention/deletion verification.
- **Sprint 6.3 — Release readiness:** Finalize dashboards, alerts, runbooks, on-call ownership, rollout/rollback, support handoff, and formal production readiness review.

### Definition of Done

- Load, reliability, security, and recovery exercises meet agreed SLOs, RTO/RPO, and tenant-isolation criteria, with evidence retained.
- Critical and high-severity findings are remediated or have explicitly approved risk acceptance and mitigation plans.
- Monitoring, alert thresholds, dashboards, runbooks, and on-call escalation paths are tested by operational scenarios.
- Deployment, rollback, migration, incident communication, and support procedures are rehearsed.
- The designated readiness authority has approved production release.

## Prioritization gates

No campaign feature before consent/suppression and delivery-status foundations. No automation before reliable inbound/outbound event processing. No CRM sync before mapping/conflict ownership is agreed. No autonomous AI messaging before governance and explicit approval.
