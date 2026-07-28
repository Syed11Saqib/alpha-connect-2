# Coding Standards

## General

Use TypeScript in strict mode. Favor explicit domain terminology, small cohesive units, immutable input DTOs, exhaustive handling of state enums, and dependency injection through interfaces/ports. Do not use `any`, implicit provider behavior, cross-module database access, or unbounded background work.

## Backend

- Controllers validate and map transport concerns only; application use cases own orchestration; domain owns business invariants; infrastructure owns Prisma/provider details.
- Every tenant-facing repository method requires a tenant scope supplied by trusted application context.
- Use transactions for related internal writes and write outbox events in the same transaction.
- Make commands idempotent where retries can occur. Set explicit external timeouts and classify retryable failures.
- Never log tokens, full message content, credentials, or sensitive PII. Use structured logs and correlation IDs.
- Database migrations are additive/backward compatible first; use expand-migrate-contract releases for destructive changes.

## Frontend

- Use React Server Components by default; use client components only for interactivity.
- Keep API authorization authoritative on the server/API; client UI permissions are presentation only.
- Model loading, empty, error, and reconnect states deliberately. Treat Socket.IO notifications as invalidation/update hints and reconcile authoritative data through APIs.
- Use shared UI primitives and accessible labels, keyboard navigation, focus handling, and color-independent status indicators.

## Security

- Validate all external input at the boundary; authorize every action and object access.
- Store secrets only through approved configuration/secret mechanisms; redact sensitive fields in logs and error reporting.
- Verify webhook signatures, protect against replay, rate-limit public endpoints, and use secure cookie/token practices.
- Review changes affecting authentication, permissions, PII, webhooks, exports, integrations, or AI prompts with a security checklist.

## Testing and review

Require unit tests for domain/application behavior, integration tests for repositories/provider adapters, contract tests for CRM and webhook payloads, and end-to-end tests for critical inbox/campaign flows. CI must run type checks, linting, tests, migration validation, dependency/security scanning, and build verification. Pull requests must state module impact, tenant/security implications, migration plan, rollback plan, and observability changes.

## Documentation discipline

Version API/event contracts intentionally. Record architectural decisions as ADRs when they affect module boundaries, data ownership, security, provider selection, or operational topology. Update these documents with any approved deviation.
