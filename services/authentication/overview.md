# mangala-authentication Service Overview

## Purpose

Central authentication and authorization service for Mangala Wallet platform. Provides passwordless authentication via WebAuthn/passkeys, JWT token management, and RBAC policy distribution.

## Technology Stack

| Component | Technology |
|-----------|------------|
| Language | Java 21 |
| Framework | Spring Boot 3.4.2 |
| Database | PostgreSQL |
| Migrations | Flyway |
| WebAuthn | WebAuthn4J 0.29.7 |
| JWT | JJWT 0.12.6 |
| Mapping | MapStruct 1.6.3 |
| Build | Gradle |

## Dependencies

### Internal
- `mangala-common-security` - Shared security models
- `mangala-exception` - Exception framework

### External
- PostgreSQL database (schema: `auth`)

## Feature Catalog

| Feature | Status | Endpoints | Key Classes |
|---------|--------|-----------|-------------|
| Passkey Registration | Done | `POST /v1/register/passkeys:start`<br>`POST /v1/register/passkeys:complete` | `StartRegistrationUseCaseImpl`<br>`CompleteRegistrationUseCaseImpl` |
| Passkey Authentication | Done | `POST /v1/authenticate/passkeys:start`<br>`POST /v1/authenticate/passkeys:complete` | `StartAuthenticationUseCaseImpl`<br>`CompleteAuthenticationUseCaseImpl` |
| Token Refresh | Done | `POST /v1/auth/refresh` | `RefreshTokenUseCaseImpl` |
| RBAC Policy Distribution | Done | `GET /v1/internal/policies`<br>`GET /v1/internal/policies/version` | `PolicyQueryServiceImpl` |

## Package Structure

```
org.mangala.authentication/
├── auth/                           # Authentication feature
│   ├── adapter/
│   │   ├── web/                    # Controllers, DTOs
│   │   └── repository/             # JPA repositories
│   ├── usecase/                    # Use case interfaces & impls
│   └── domain/                     # Entities, exceptions
│
├── passkey/                        # Passkey/WebAuthn feature
│   ├── adapter/
│   │   └── repository/
│   ├── usecase/
│   └── domain/
│
├── user/                           # User management
│   ├── adapter/
│   │   └── repository/
│   ├── usecase/
│   └── domain/
│
├── policy/                         # Internal policy API
│   ├── adapter/
│   │   └── web/
│   └── usecase/
│
└── shared/                         # Shared configuration
    ├── config/
    └── exception/
```

## Database Schema

| Table | Purpose |
|-------|---------|
| `users` | User accounts |
| `user_passkeys` | WebAuthn credentials |
| `passkey_challenges` | Authentication challenges |
| `refresh_tokens` | JWT refresh token tracking |
| `permissions` | Permission definitions |
| `roles` | Role definitions |
| `role_permissions` | Role-permission mapping |
| `user_roles` | User-role mapping |
| `api_permissions` | API authorization policies |
| `policy_version` | Policy cache version |

## Configuration

### Required Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `DB_HOST` | PostgreSQL host | - |
| `DB_PORT` | PostgreSQL port | 5432 |
| `DB_NAME` | Database name | - |
| `DB_SCHEMA` | Schema name | auth |
| `DB_USERNAME` | Database user | - |
| `DB_PASSWORD` | Database password | - |
| `JWT_SECRET` | HMAC signing key (256-bit min) | - |
| `JWT_ISSUER` | Token issuer | mangala |
| `JWT_ACCESS_EXPIRATION_SECONDS` | Access token TTL | 900 |
| `JWT_REFRESH_EXPIRATION_SECONDS` | Refresh token TTL | 604800 |
| `RP_ID` | WebAuthn relying party ID | - |
| `RP_NAME` | WebAuthn relying party name | - |
| `RP_ORIGIN` | WebAuthn origin URL | - |
| `INTERNAL_API_MTLS_ENABLED` | Enable mTLS for internal APIs | false |

## Related Documentation

- **Design**: `design.md` - Architecture, flows, data model
- **API**: `api.md` - Endpoint specifications
- **ADRs**:
  - [ADR-0001: Policy Loading Via Auth Internal API](../../adr/README.md)
  - [ADR-0002: Internal Policy mTLS Authorization](../../adr/README.md)
