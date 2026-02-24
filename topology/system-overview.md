# Mangala System Overview

## System Purpose

Mangala is a digital wallet platform providing secure authentication and authorization for financial services.

## Services Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Clients                                  │
│              (Web App, Mobile App, API Consumers)                │
└─────────────────────────────────────┬───────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                      mangala-gateway                             │
│  • API Gateway / Reverse Proxy                                   │
│  • JWT Validation                                                │
│  • Authorization Enforcement (RBAC/ABAC)                         │
│  • Rate Limiting                                                 │
│  • Request Routing                                               │
└─────────────────────────────────────┬───────────────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                 │
                    ▼                 ▼                 ▼
┌───────────────────────┐ ┌───────────────────┐ ┌───────────────────┐
│ mangala-authentication│ │   mangala-wallet  │ │  (future services)│
│                       │ │    (planned)      │ │                   │
│ • User Registration   │ │ • Wallet Mgmt     │ │                   │
│ • Passkey Auth        │ │ • Transactions    │ │                   │
│ • JWT Issuance        │ │ • Balances        │ │                   │
│ • Token Refresh       │ │                   │ │                   │
│ • RBAC Policies       │ │                   │ │                   │
│ • Internal Policy API │ │                   │ │                   │
└───────────┬───────────┘ └─────────┬─────────┘ └───────────────────┘
            │                       │
            ▼                       ▼
┌───────────────────────┐ ┌───────────────────┐
│   PostgreSQL (auth)   │ │ PostgreSQL (wallet)│
│   Schema: auth        │ │ Schema: wallet    │
└───────────────────────┘ └───────────────────┘
```

## Services

### mangala-gateway

**Purpose**: API Gateway that handles all external traffic

**Responsibilities**:
- Route requests to appropriate backend services
- Validate JWT access tokens
- Enforce authorization policies (loaded from auth service)
- Rate limiting and throttling
- Request/response transformation

**Dependencies**:
- mangala-authentication (internal policy API)
- mangala-common-security (shared models)

### mangala-authentication

**Purpose**: Central authentication and authorization service

**Responsibilities**:
- User registration with WebAuthn/passkeys
- Passkey-based authentication
- JWT token issuance (access + refresh tokens)
- Token refresh with rotation
- RBAC policy management
- Internal policy API for gateway

**Dependencies**:
- PostgreSQL (auth schema)
- mangala-common-security (shared models)
- mangala-exception (error handling)

**Provides To**:
- Gateway: JWT tokens, authorization policies

### mangala-wallet (planned)

**Purpose**: Wallet and transaction management

**Responsibilities**:
- Wallet creation and management
- Transaction processing
- Balance tracking

## Shared Libraries

### mangala-common-security

**Purpose**: Shared security models and utilities used by multiple services

**Contents**:
- `ApiPermissionDTO` - Policy rule model
- `JwtUserClaims` - JWT claim structure
- `PolicyRule` - Policy evaluation model
- `AuthorizationResult` - Auth decision result
- `PolicyUpdateEvent` - Async policy events
- `PermissionMatcher` - Permission matching
- `PathNormalizer` - URL normalization
- `SecurityConstants` - Shared constants

**Used By**: mangala-authentication, mangala-gateway

### mangala-exception

**Purpose**: Common exception handling framework

**Used By**: All services

## Communication Patterns

### External → Gateway

| Protocol | Description |
|----------|-------------|
| HTTPS | All external API calls |
| WebAuthn | Browser passkey ceremonies |

### Gateway → Backend Services

| From | To | Protocol | Purpose |
|------|-----|----------|---------|
| Gateway | Authentication | HTTPS (mTLS) | Policy loading |
| Gateway | Authentication | HTTPS | Token validation (via JWT signature) |
| Gateway | Wallet | HTTPS | Wallet operations |

### Service → Database

| Service | Database | Schema |
|---------|----------|--------|
| mangala-authentication | PostgreSQL | auth |
| mangala-wallet | PostgreSQL | wallet |

## Security Model

### External Security
- HTTPS for all external traffic
- JWT Bearer tokens for API authentication
- WebAuthn/FIDO2 for passwordless authentication

### Internal Security
- mTLS for service-to-service communication (configurable)
- x509 client certificates identify services
- Internal APIs protected by certificate allowlist

### Authorization Model
- **RBAC**: Role-Based Access Control
  - Users → Roles → Permissions
- **ABAC**: Attribute-Based Access Control (optional)
  - SpEL condition expressions on policies
- Policies loaded by gateway from auth service
- Gateway enforces authorization at request time

## Technology Stack

| Layer | Technology |
|-------|------------|
| Language | Java 21 |
| Framework | Spring Boot 3.4.x |
| Database | PostgreSQL |
| Migrations | Flyway |
| Auth | WebAuthn4J, JJWT |
| Build | Gradle |
| Testing | JUnit 5, Testcontainers |

## Configuration

### Environment Variables (Authentication Service)

| Variable | Purpose |
|----------|---------|
| `DB_HOST`, `DB_PORT`, `DB_NAME` | Database connection |
| `DB_USERNAME`, `DB_PASSWORD` | Database credentials |
| `DB_SCHEMA` | Database schema (default: auth) |
| `JWT_SECRET` | HMAC signing key (256-bit min) |
| `JWT_ISSUER` | Token issuer identifier |
| `JWT_ACCESS_EXPIRATION_SECONDS` | Access token TTL |
| `JWT_REFRESH_EXPIRATION_SECONDS` | Refresh token TTL |
| `RP_ID`, `RP_NAME`, `RP_ORIGIN` | WebAuthn relying party |
| `INTERNAL_API_MTLS_ENABLED` | Enable mTLS for internal APIs |
| `INTERNAL_API_ALLOWED_PRINCIPALS` | Allowed certificate CNs |
