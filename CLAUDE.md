# Mangala Architecture - AI Context

This is the central documentation repository for all Mangala microservices. It serves as persistent AI context to maintain consistency across Claude Code sessions.

## Quick Context Load (read in order)

When starting work on any Mangala service, read these files first:

1. `topology/system-overview.md` - Understand the overall system architecture
2. `shared/common-security.md` - Shared security contracts and models
3. `patterns/architecture.md` - Coding patterns and conventions to follow

## Service-Specific Context

When working on a specific service, also read:

- `services/{service}/overview.md` - Features and current status
- `services/{service}/design.md` - Architecture decisions and data flows
- `services/{service}/api.md` - API contracts and schemas

### Available Services

| Service | Directory | Status | Purpose |
|---------|-----------|--------|---------|
| Authentication | `services/authentication/` | ✅ Complete | User authentication, JWT tokens, RBAC policies |
| Gateway | `services/gateway/` | ✅ Complete | API routing, rate limiting, authorization |
| Wallet | `services/wallet/` | 📋 PRD Ready | Wallet tracking, balance sync |
| Portfolio | `services/portfolio/` | 📋 PRD Ready | Portfolio aggregation, P&L, real-time |

## Product Requirements (PRDs)

For planned/in-progress services, read PRDs first:

- `prd/wallet-service.md` - Wallet service requirements
- `prd/portfolio-service.md` - Portfolio service requirements
- `prd/task-breakdown.md` - Implementation tasks and progress

## Cross-Service Changes

For changes affecting multiple services, additionally read:

- `shared/contracts.md` - Inter-service API contracts
- `shared/error-codes.md` - Error code registry (prevents collisions)
- `adr/README.md` - Architecture decision records

## Patterns to Follow

All code must follow patterns documented in: `patterns/architecture.md`

Key patterns:
- Feature-first layered architecture
- Use case pattern (interface + implementation)
- Command/Query separation
- MapStruct for DTO mapping
- Flyway for database migrations

## Repository Layout

```
mangala-architecture/
├── CLAUDE.md                    # This file - AI context manifest
├── README.md                    # Human overview
├── glossary.md                  # Domain terminology
├── topology/
│   └── system-overview.md       # System architecture
├── shared/
│   ├── common-security.md       # mangala-common-security docs
│   ├── error-codes.md           # Cross-service error codes
│   └── contracts.md             # Inter-service contracts
├── services/
│   └── {service}/
│       ├── overview.md          # Feature catalog
│       ├── design.md            # Architecture & flows
│       ├── api.md               # API documentation
│       └── context-map.yaml     # Service context (synced)
├── patterns/
│   └── architecture.md          # All coding patterns
└── adr/
    ├── README.md                # ADR index
    └── template.md              # ADR template
```

## How to Use This Repository

1. **Before coding**: Read the Quick Context Load files
2. **Service-specific work**: Read that service's directory
3. **Cross-service work**: Also read shared/ and adr/
4. **After implementing**: Update relevant docs if patterns/contracts change
