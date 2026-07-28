# Folder Structure

## Monorepo layout

```text
.
├── apps/
│   ├── web/                         # Next.js 15 tenant/admin application
│   ├── api/                         # NestJS HTTP + Socket.IO application
│   └── worker/                      # NestJS queued/scheduled processors
├── packages/
│   ├── contracts/                   # Versioned DTO/event schemas; no framework imports
│   ├── config/                      # Typed, validated configuration contracts
│   ├── ui/                          # Shared React/Tailwind/shadcn-ui primitives
│   ├── eslint-config/
│   ├── typescript-config/
│   └── test-utils/
├── prisma/
│   ├── schema.prisma
│   ├── migrations/
│   └── seed/                        # Non-production fixture strategy only
├── docs/
├── infra/                           # Docker and deployment manifests; no application code
├── scripts/                         # Operational scripts
└── .github/                         # CI/CD workflows and templates
```

## Complete example project tree

The following is an illustrative target layout. It documents ownership and placement; it is not a directive to scaffold these folders before implementation is approved.

```text
.
├── apps/
│   ├── api/
│   │   ├── src/
│   │   │   ├── bootstrap/
│   │   │   ├── common/
│   │   │   │   ├── auth/
│   │   │   │   ├── database/
│   │   │   │   ├── events/
│   │   │   │   ├── errors/
│   │   │   │   ├── observability/
│   │   │   │   ├── rate-limiting/
│   │   │   │   └── tenancy/
│   │   │   ├── modules/
│   │   │   │   ├── identity-access/
│   │   │   │   ├── tenant-management/
│   │   │   │   ├── contacts/
│   │   │   │   ├── inbox/
│   │   │   │   ├── templates/
│   │   │   │   ├── campaigns/
│   │   │   │   ├── chatbot-builder/
│   │   │   │   ├── notifications/
│   │   │   │   ├── whatsapp-provider/
│   │   │   │   ├── integration-layer/
│   │   │   │   ├── crm-integration/
│   │   │   │   ├── ai-assistance/
│   │   │   │   ├── analytics/
│   │   │   │   └── media/
│   │   │   ├── realtime/
│   │   │   └── main.ts
│   │   └── test/
│   ├── web/
│   │   ├── app/
│   │   │   ├── (public)/
│   │   │   ├── (auth)/
│   │   │   ├── (tenant)/
│   │   │   └── api/                  # Next.js-only BFF/proxy routes when justified
│   │   ├── features/
│   │   │   ├── inbox/
│   │   │   ├── contacts/
│   │   │   ├── templates/
│   │   │   ├── campaigns/
│   │   │   ├── automation/
│   │   │   ├── analytics/
│   │   │   ├── integrations/
│   │   │   ├── settings/
│   │   │   └── shared/
│   │   ├── components/               # Web-specific composition only
│   │   ├── lib/                      # API client, auth/session, realtime client
│   │   ├── hooks/
│   │   └── styles/
│   └── worker/
│       ├── src/
│       │   ├── bootstrap/
│       │   ├── processors/           # Campaign, CRM, media, AI, projection processors
│       │   ├── schedulers/
│       │   └── main.ts
│       └── test/
├── packages/
│   ├── contracts/                    # Versioned REST, Socket.IO, event, job schemas
│   ├── database/                     # Prisma client, migrations/schema ownership, repositories support
│   ├── ui/                           # Shared React, Tailwind, shadcn/ui primitives
│   ├── config/                       # Typed configuration and environment validation
│   ├── eslint-config/
│   ├── typescript-config/
│   └── test-utils/
├── docs/
├── infra/
│   ├── docker/
│   ├── deployment/
│   ├── observability/
│   └── secrets/                      # References/templates only; never plaintext production secrets
├── scripts/
│   ├── database/
│   ├── operations/
│   └── verification/
├── .github/
│   ├── workflows/
│   └── ISSUE_TEMPLATE/
├── prisma/                           # If retained at root; otherwise owned by packages/database
├── package.json
├── workspace configuration
└── Docker configuration
```

Within each `apps/api/src/modules/<module>` directory, use the `domain`, `application`, `infrastructure`, and `presentation` layout described below. `apps/worker` hosts independent worker entry points and processors, but reuses application modules and contracts; it must not duplicate domain business rules or import API presentation layers.

## API module layout

```text
apps/api/src/
├── bootstrap/                       # Application startup, global middleware
├── common/                          # Cross-cutting primitives only
│   ├── auth/ ├── database/ ├── observability/ ├── events/ └── errors/
├── modules/
│   └── <module>/
│       ├── domain/                  # Entities, value objects, domain events, ports
│       ├── application/             # Use cases, commands/queries, DTO mapping
│       ├── infrastructure/          # Prisma repositories, provider adapters
│       ├── presentation/            # REST/websocket controllers, guards
│       └── <module>.module.ts
└── main.ts
```

## Dependency direction

`presentation -> application -> domain`; `infrastructure` implements ports defined by `domain` or `application`. `common` may not depend on business modules. Module-to-module usage goes through exported application interfaces or contracts only. The worker imports application modules, not presentation layers.

## Web application layout

Use route groups for authenticated tenant UI and public/auth pages. Organize features by domain (inbox, contacts, campaigns) with colocated screens, hooks, API clients, and tests. Shared UI belongs in `packages/ui`; business authorization remains server/API enforced.

## Naming and boundaries

Use plural REST resources, singular domain entities, explicit command/query names, and provider-specific adapters named by provider. Avoid generic `utils`, `helpers`, and `services` directories; locate code in its owning module.
