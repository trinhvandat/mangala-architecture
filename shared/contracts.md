# Inter-Service Contracts

This document defines the API contracts between Mangala services. Changes to these contracts require coordination between teams.

## Gateway ↔ Authentication

### Contract: Internal Policy API

**Purpose**: Gateway loads authorization policies from authentication service.

**Provider**: mangala-authentication
**Consumer**: mangala-gateway

#### GET /v1/internal/policies

Returns all active API authorization policies.

**Security**: mTLS (when `internal-api.mtls-enabled=true`)
- Requires x509 client certificate
- Certificate CN must be in `allowed-principals` list
- Grants `INTERNAL_POLICY_READ` authority

**Response**: `200 OK`
```json
[
  {
    "id": "f56f7df6-4ee6-4f0f-a3a0-f6f45e90ea6d",
    "httpMethod": "GET",
    "pathPattern": "/api/v1/wallets",
    "permissionCode": "wallets:read",
    "conditionExpr": null,
    "serviceName": "wallet-service",
    "priority": 10,
    "active": true
  }
]
```

**Model**: `org.mangala.security.model.ApiPermissionDTO`

#### GET /v1/internal/policies/version

Returns current policy version for cache invalidation.

**Security**: Same as /policies

**Response**: `200 OK`
```json
42
```

**Usage Pattern**:
1. Gateway starts → calls `/v1/internal/policies` → loads cache
2. Gateway polls `/v1/internal/policies/version` periodically
3. If version changed → reload full policy set
4. Atomic swap of in-memory cache

**Sequence Diagram**:
```
┌─────────┐                    ┌─────────────────┐                    ┌────────┐
│ Gateway │                    │ Authentication  │                    │   DB   │
└────┬────┘                    └────────┬────────┘                    └───┬────┘
     │                                  │                                 │
     │ GET /v1/internal/policies        │                                 │
     │ [mTLS client cert]               │                                 │
     │─────────────────────────────────>│                                 │
     │                                  │ SELECT policies                 │
     │                                  │────────────────────────────────>│
     │                                  │ <───────────────────────────────│
     │ <────────────────────────────────│                                 │
     │ 200 OK [List<ApiPermissionDTO>]  │                                 │
     │                                  │                                 │
     │ Load into cache                  │                                 │
     │                                  │                                 │
```

---

### Contract: JWT Token Format

**Purpose**: Gateway validates JWTs issued by authentication service.

**Provider**: mangala-authentication
**Consumer**: mangala-gateway

#### Access Token Claims

```json
{
  "iss": "mangala",
  "sub": "550e8400-e29b-41d4-a716-446655440000",
  "aud": "mangala-gateway",
  "iat": 1705312200,
  "exp": 1705313100,
  "jti": "unique-token-id",
  "type": "access",
  "email": "user@example.com",
  "roles": ["ROLE_USER"],
  "permissions": ["wallets:read", "wallets:create"],
  "amr": ["passkey"]
}
```

| Claim | Type | Description |
|-------|------|-------------|
| `iss` | string | Issuer identifier |
| `sub` | string | User UUID |
| `aud` | string | Intended audience |
| `iat` | number | Issued at (Unix timestamp) |
| `exp` | number | Expiration (Unix timestamp) |
| `jti` | string | Unique token ID |
| `type` | string | Token type (`access` or `refresh`) |
| `email` | string | User email (optional) |
| `roles` | array | Role codes |
| `permissions` | array | Permission codes |
| `amr` | array | Authentication methods |

#### Refresh Token Claims

```json
{
  "iss": "mangala",
  "sub": "550e8400-e29b-41d4-a716-446655440000",
  "aud": "mangala-auth-refresh",
  "iat": 1705312200,
  "exp": 1705917000,
  "jti": "refresh-token-id",
  "type": "refresh",
  "email": "user@example.com"
}
```

**Note**: Refresh tokens do NOT contain roles/permissions (they're fetched fresh on refresh).

#### Token Validation (Gateway)

1. Extract `Authorization: Bearer <token>` header
2. Verify JWT signature using shared secret
3. Check `aud` claim matches expected audience
4. Check `exp` > current time
5. Check `type` = `access`
6. Extract user claims for authorization

---

### Contract: Token Refresh

**Purpose**: Client refreshes expired access token.

**Provider**: mangala-authentication
**Consumer**: Client applications (via gateway)

#### POST /v1/auth/refresh

**Request**:
```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIs..."
}
```

**Response**: `200 OK`
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
  "tokenType": "Bearer",
  "expiresIn": 900,
  "refreshExpiresIn": 604800,
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com"
}
```

**Behavior**:
- Old refresh token is revoked (single-use)
- New refresh token is issued
- Roles/permissions are fetched fresh from database

---

## Gateway ↔ Downstream Services

### Contract: User Context Headers

**Purpose**: Gateway forwards authenticated user context to downstream services.

**Provider**: mangala-gateway
**Consumer**: All downstream services

When a request is authorized, gateway adds these headers:

| Header | Value | Example |
|--------|-------|---------|
| `X-User-Id` | User UUID | `550e8400-e29b-41d4-a716-446655440000` |
| `X-User-Email` | User email | `user@example.com` |
| `X-User-Roles` | Comma-separated roles | `ROLE_USER,ROLE_ADMIN` |
| `X-User-Permissions` | Comma-separated permissions | `wallets:read,wallets:create` |
| `X-Request-Id` | Request trace ID | `abc-123-def` |
| `X-Correlation-Id` | Correlation ID | `xyz-789` |

**Trust Model**: Downstream services MUST trust these headers when request comes from gateway. They should NOT re-validate the JWT.

---

## Contract Versioning

### Breaking Changes

A breaking change is:
- Removing a field
- Changing a field's type
- Changing required/optional status
- Changing authentication requirements

### Non-Breaking Changes

- Adding optional fields
- Adding new endpoints
- Adding new enum values (if consumers handle unknown values)

### Coordination Process

1. Provider announces deprecation in ADR
2. Minimum 2-week notice for breaking changes
3. Consumer updates first, then provider deploys
4. For critical fixes: coordinate same-day deployment
