# Mangala Architecture

Central documentation repository for all Mangala microservices.

## Purpose

This repository serves two primary goals:

1. **AI Context Persistence** - Provides Claude Code with consistent context across sessions via `CLAUDE.md`
2. **Cross-Service Documentation** - Documents patterns, contracts, and decisions that span multiple services

## For AI Assistants

Start by reading `CLAUDE.md` which provides structured instructions for loading context.

## For Developers

| Document | Purpose |
|----------|---------|
| `topology/system-overview.md` | System architecture and service relationships |
| `shared/common-security.md` | Shared security library documentation |
| `shared/contracts.md` | Inter-service API contracts |
| `patterns/architecture.md` | Coding patterns and conventions |
| `services/{name}/` | Service-specific documentation |
| `adr/` | Architecture Decision Records |
| `glossary.md` | Domain terminology |

## Services Documented

- **Authentication** (`services/authentication/`) - User authentication, JWT tokens, RBAC
- **Wallet** (`services/wallet/`) - Wallet management, balance sync, multi-chain EVM support

## Contributing

When making changes to Mangala services:

1. Update relevant service documentation if APIs or architecture change
2. Add ADRs for significant architectural decisions
3. Keep `context-map.yaml` synced from service repos
4. Update error codes registry when adding new error codes
