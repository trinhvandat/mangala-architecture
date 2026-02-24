# Product Requirements Documents

This directory contains PRDs for all Mangala services.

## Document Index

| Document | Service | Status | Last Updated |
|----------|---------|--------|--------------|
| [wallet-service.md](./wallet-service.md) | mangala-wallet | Draft | 2024-02-24 |
| [portfolio-service.md](./portfolio-service.md) | mangala-portfolio | Draft | 2024-02-24 |
| [task-breakdown.md](./task-breakdown.md) | All Services | Active | 2024-02-24 |

## Existing Services (Implemented)

| Service | Status | Documentation |
|---------|--------|---------------|
| mangala-authentication | ✅ Complete | [services/authentication/](../services/authentication/) |
| mangala-gateway | ✅ Complete | [services/gateway/](../services/gateway/) (pending) |

## Planned Services

| Service | PRD Status | Target Sprint |
|---------|------------|---------------|
| mangala-wallet | Draft Complete | Sprint 1-2 |
| mangala-portfolio | Draft Complete | Sprint 2-3 |
| Price Feed Service | Included in Portfolio | Sprint 2 |
| Blockchain Crawler | Not Started | Future |
| Transaction History | Not Started | Future |
| Notification Service | Not Started | Future |

## Document Template

When creating a new PRD, include:

1. **Executive Summary** - Vision, objectives, target users
2. **Problem Statement** - Pain points and solution
3. **User Stories** - Prioritized with acceptance criteria
4. **Functional Requirements** - Detailed specifications
5. **Non-Functional Requirements** - Performance, reliability, security
6. **API Specification** - Endpoints, request/response formats
7. **Data Model** - Database schemas, collections
8. **Dependencies** - Internal and external
9. **Milestones** - Delivery timeline
10. **Open Questions** - Items needing resolution

## Related Documents

- [System Overview](../topology/system-overview.md)
- [Coding Patterns](../patterns/architecture.md)
- [Shared Contracts](../shared/contracts.md)
- [Error Codes](../shared/error-codes.md)
