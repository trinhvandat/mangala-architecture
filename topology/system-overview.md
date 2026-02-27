# Mangala System Overview

## System Purpose

Mangala is a digital wallet platform providing secure authentication, wallet tracking, and portfolio management for crypto assets across multiple EVM-compatible blockchains.

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
          ┌───────────────────────────┼───────────────────────────┐
          │                           │                           │
          ▼                           ▼                           ▼
┌───────────────────────┐ ┌───────────────────────┐ ┌───────────────────────┐
│ mangala-authentication│ │   mangala-wallet      │ │  mangala-portfolio    │
│                       │ │                       │ │                       │
│ • User Registration   │ │ • Wallet CRUD         │ │ • Portfolio CRUD      │
│ • Passkey Auth        │ │ • Balance Tracking    │ │ • Holdings Aggregation│
│ • JWT Issuance        │ │ • EVM Chain Support   │ │ • History/Snapshots   │
│ • Token Refresh       │ │ • Balance Sync        │ │ • Multi-wallet View   │
│ • RBAC Policies       │ │ • Kafka Events        │ │ • Kafka Consumers     │
│ • Internal Policy API │ │                       │ │                       │
└───────────┬───────────┘ └───────────┬───────────┘ └───────────┬───────────┘
            │                         │                         │
            ▼                         ▼                         ▼
┌───────────────────────┐ ┌───────────────────────┐ ┌───────────────────────┐
│   PostgreSQL (auth)   │ │ PostgreSQL (wallet)   │ │ PostgreSQL (portfolio)│
│   Schema: auth        │ │ + Redis (cache)       │ │ + Redis (cache)       │
└───────────────────────┘ └───────────────────────┘ └───────────────────────┘
                                      │                         │
                                      └────────────┬────────────┘
                                                   ▼
                                          ┌───────────────┐
                                          │     Kafka     │
                                          │ • wallet.balance.updated │
                                          │ • portfolio.updated      │
                                          └───────────────┘
```

## Services

### mangala-gateway

**Purpose**: API Gateway that handles all external traffic

**Port**: 8000

**Responsibilities**:
- Route requests to appropriate backend services
- Validate JWT access tokens
- Enforce authorization policies (loaded from auth service)
- Rate limiting and throttling
- Request/response transformation
- Circuit breaker for fault tolerance

**Dependencies**:
- mangala-authentication (internal policy API)
- mangala-common-security (shared models)
- Redis (rate limiting)

### mangala-authentication

**Purpose**: Central authentication and authorization service

**Port**: 8080

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

### mangala-wallet

**Purpose**: Wallet tracking and balance management for EVM chains

**Port**: 8081

**Responsibilities**:
- Wallet CRUD operations (add/remove watched addresses)
- Balance tracking (native + ERC-20 tokens)
- Multi-chain support (Ethereum, BSC, Polygon, Arbitrum)
- Balance synchronization with blockchain
- Publish balance update events to Kafka

**Dependencies**:
- PostgreSQL (wallet schema)
- Redis (balance cache, rate limiting)
- Kafka (event publishing)
- Web3j (blockchain interaction)
- mangala-common-security (shared models)
- mangala-exception (error handling)

**API Endpoints**:
| Method | Path | Description |
|--------|------|-------------|
| POST | /api/v1/wallets | Create wallet |
| GET | /api/v1/wallets | List user's wallets |
| GET | /api/v1/wallets/{id} | Get wallet details |
| DELETE | /api/v1/wallets/{id} | Delete wallet |
| GET | /api/v1/wallets/{id}/balances | Get wallet balances |
| POST | /api/v1/wallets/{id}/sync | Trigger balance sync |

### mangala-portfolio

**Purpose**: Portfolio aggregation and holdings management

**Port**: 8082

**Responsibilities**:
- Portfolio CRUD operations
- Aggregate holdings across multiple wallets
- Calculate total portfolio value
- Track historical portfolio values (snapshots)
- Real-time updates via Kafka consumers

**Dependencies**:
- PostgreSQL (portfolio schema)
- Redis (aggregation cache)
- Kafka (event consumption)
- mangala-wallet (wallet data via API)
- mangala-common-security (shared models)
- mangala-exception (error handling)

**API Endpoints**:
| Method | Path | Description |
|--------|------|-------------|
| POST | /api/v1/portfolios | Create portfolio |
| GET | /api/v1/portfolios | List portfolios |
| GET | /api/v1/portfolios/{id} | Get portfolio |
| PUT | /api/v1/portfolios/{id} | Update portfolio |
| DELETE | /api/v1/portfolios/{id} | Delete portfolio |
| POST | /api/v1/portfolios/{id}/wallets | Add wallets |
| DELETE | /api/v1/portfolios/{id}/wallets/{walletId} | Remove wallet |
| GET | /api/v1/portfolios/{id}/summary | Get portfolio summary |
| GET | /api/v1/portfolios/{id}/holdings | Get holdings |
| GET | /api/v1/portfolios/{id}/history | Get history |
| POST | /api/v1/portfolios/{id}/sync | Trigger recalculation |

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

**Used By**: mangala-authentication, mangala-gateway, mangala-wallet, mangala-portfolio

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
| Gateway | Portfolio | HTTPS | Portfolio operations |

### Service → Service (Async)

| From | To | Protocol | Topic |
|------|-----|----------|-------|
| Wallet | Portfolio | Kafka | wallet.balance.updated |
| Portfolio | Clients | Kafka | portfolio.updated |

### Service → Database

| Service | Database | Schema | Cache |
|---------|----------|--------|-------|
| mangala-authentication | PostgreSQL | auth | - |
| mangala-wallet | PostgreSQL | wallet | Redis |
| mangala-portfolio | PostgreSQL | portfolio | Redis |

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

### Permissions

| Permission | Description | Roles |
|------------|-------------|-------|
| wallets:create | Create wallet | USER, PREMIUM, ADMIN |
| wallets:read | View wallets | USER, PREMIUM, ADMIN |
| wallets:update | Update wallet | PREMIUM, ADMIN |
| wallets:delete | Delete wallet | PREMIUM, ADMIN |
| wallets:sync | Sync wallet | USER, PREMIUM, ADMIN |
| portfolios:create | Create portfolio | USER, PREMIUM, ADMIN |
| portfolios:read | View portfolios | USER, PREMIUM, ADMIN |
| portfolios:update | Update portfolio | USER, PREMIUM, ADMIN |
| portfolios:delete | Delete portfolio | PREMIUM, ADMIN |
| holdings:read | View holdings | USER, PREMIUM, ADMIN |

## Technology Stack

| Layer | Technology |
|-------|------------|
| Language | Java 21 |
| Framework | Spring Boot 3.4.x |
| Database | PostgreSQL 15+ |
| Cache | Redis 7+ |
| Messaging | Apache Kafka |
| Blockchain | Web3j 4.x |
| Migrations | Flyway |
| Auth | WebAuthn4J, JJWT |
| Build | Maven / Gradle |
| Testing | JUnit 5, Testcontainers, H2 |
| Container | Docker |
| Orchestration | Kubernetes |
| CI/CD | GitHub Actions |

## Configuration

### Environment Variables (Common)

| Variable | Purpose |
|----------|---------|
| `DB_HOST`, `DB_PORT`, `DB_NAME` | Database connection |
| `DB_USERNAME`, `DB_PASSWORD` | Database credentials |
| `REDIS_HOST`, `REDIS_PORT` | Redis connection |
| `KAFKA_BOOTSTRAP_SERVERS` | Kafka brokers |

### Environment Variables (Authentication Service)

| Variable | Purpose |
|----------|---------|
| `DB_SCHEMA` | Database schema (default: auth) |
| `JWT_SECRET` | HMAC signing key (256-bit min) |
| `JWT_ISSUER` | Token issuer identifier |
| `JWT_ACCESS_EXPIRATION_SECONDS` | Access token TTL |
| `JWT_REFRESH_EXPIRATION_SECONDS` | Refresh token TTL |
| `RP_ID`, `RP_NAME`, `RP_ORIGIN` | WebAuthn relying party |
| `INTERNAL_API_MTLS_ENABLED` | Enable mTLS for internal APIs |
| `INTERNAL_API_ALLOWED_PRINCIPALS` | Allowed certificate CNs |

### Environment Variables (Wallet Service)

| Variable | Purpose |
|----------|---------|
| `ETHEREUM_RPC_URL` | Ethereum node RPC |
| `BSC_RPC_URL` | BSC node RPC |
| `POLYGON_RPC_URL` | Polygon node RPC |
| `ARBITRUM_RPC_URL` | Arbitrum node RPC |
| `SYNC_INTERVAL_SECONDS` | Balance sync interval |

### Environment Variables (Portfolio Service)

| Variable | Purpose |
|----------|---------|
| `WALLET_SERVICE_URL` | Wallet service base URL |
| `SNAPSHOT_CRON` | Snapshot schedule (cron) |
| `CACHE_TTL_SECONDS` | Holdings cache TTL |

## Deployment

### Kubernetes Resources

Each service includes:
- Deployment (2+ replicas)
- Service (ClusterIP)
- ConfigMap (configuration)
- Secrets (credentials)
- HorizontalPodAutoscaler (2-10 replicas, 70% CPU)

### CI/CD Pipeline

GitHub Actions workflow:
1. Build (Maven/Gradle)
2. Test (unit + integration)
3. Docker build
4. Push to ghcr.io
5. Deploy to Kubernetes
