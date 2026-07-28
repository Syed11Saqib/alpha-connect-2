# Implementation Guide

## Purpose

This guide defines how implementation must be carried out. The architecture documents inside `/docs` are the source of truth. If implementation conflicts with the architecture, implementation must change.

---

## Source of Truth

Always follow these documents:

- docs/architecture.md
- docs/modules.md
- docs/database-design.md
- docs/api-contracts.md
- docs/folder-structure.md
- docs/integrations.md
- docs/roadmap.md
- docs/coding-standards.md

---

## General Rules

- Implement one sprint or one module at a time.
- Never implement multiple major modules in one task.
- Never make architectural assumptions.
- Stop and ask for clarification if requirements are ambiguous.
- Keep commits small and focused.

---

## Clean Architecture

Always follow:

Presentation
↓
Application
↓
Domain
↓
Infrastructure

- Controllers contain no business logic.
- Domain contains business rules.
- Infrastructure implements interfaces.
- Never bypass architecture layers.

---

## Module Communication

- Modules communicate only through exported interfaces, contracts, or events.
- Never access another module's database directly.

---

## Database

- Use Prisma.
- Use migrations only.
- Every tenant-owned entity must be tenant-scoped.
- Follow expand → migrate → contract for schema evolution.

---

## API

- Follow api-contracts.md.
- Validate every request.
- Use standardized error responses.
- Use correlation IDs.
- Use idempotency where required.

---

## Security

- Validate every external input.
- Authorize every action.
- Never log secrets or sensitive data.
- Verify webhook signatures.
- Encrypt credentials.

---

## Testing

Every implementation must include:

- Unit tests
- Integration tests
- End-to-end tests where applicable

---

## Documentation

If implementation changes APIs, schemas, or architecture:

- Update documentation.
- Update OpenAPI.
- Update ADRs if architecture changes.

---

## Definition of Done

A task is complete only if:

- Code compiles.
- Tests pass.
- Lint passes.
- Documentation is updated.
- Architecture rules are respected.

---

## Workflow

Always work in this order:

1. Plan
2. Review
3. Implement
4. Test
5. Review
6. Commit
