# mangala-authentication API

## Base URL

`/v1` - All endpoints are versioned under v1.

## Public Endpoints

### Passkey Registration

#### POST /v1/register/passkeys:start

Initiates WebAuthn passkey registration ceremony.

**Request Headers**:
| Header | Required | Description |
|--------|----------|-------------|
| `User-Agent` | Yes | Browser/device info (extracted for device name) |
| `X-Forwarded-For` | No | Client IP address |

**Request Body**:
```json
{
  "email": "user@example.com"  // Optional
}
```

**Response**: `200 OK`
```json
{
  "sessionId": "abc123def456",
  "rp": {
    "id": "example.com",
    "name": "Mangala Wallet"
  },
  "user": {
    "id": "base64url-encoded-user-id",
    "name": "user@example.com",
    "displayName": "user@example.com"
  },
  "challenge": "base64url-encoded-challenge",
  "pubKeyCredParams": [
    {"type": "public-key", "alg": -7},
    {"type": "public-key", "alg": -257}
  ],
  "timeout": 60000,
  "excludeCredentials": [],
  "authenticatorSelection": {
    "authenticatorAttachment": "platform",
    "residentKey": "required",
    "userVerification": "preferred"
  },
  "attestation": "none"
}
```

**Errors**:
| Code | HTTP | Description |
|------|------|-------------|
| `0000001` | 409 | User with email already exists |

---

#### POST /v1/register/passkeys:complete

Completes WebAuthn registration with credential attestation.

**Request Body**:
```json
{
  "sessionId": "abc123def456",
  "credential": {
    "id": "base64url-credential-id",
    "response": {
      "clientDataJSON": "base64url-encoded",
      "attestationObject": "base64url-encoded",
      "transports": ["internal", "hybrid"]
    }
  },
  "authenticatorAttachment": "platform"
}
```

**Response**: `200 OK`
```json
{
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "credentialId": "base64url-credential-id",
  "userVerified": true,
  "backupEligible": true,
  "backupState": false,
  "deviceName": "Chrome on macOS",
  "createdAt": "2024-01-15T10:30:00Z"
}
```

**Errors**:
| Code | HTTP | Description |
|------|------|-------------|
| `0000002` | 404 | Challenge not found |
| `0000003` | 400 | Challenge expired |
| `0000004` | 400 | Challenge already used |
| `0000005` | 400 | Credential verification failed |
| `0000006` | 409 | Duplicate credential ID |
| `0000007` | 400 | Origin validation failed |
| `0000008` | 400 | Attestation verification failed |

---

### Passkey Authentication

#### POST /v1/authenticate/passkeys:start

Initiates WebAuthn authentication ceremony.

**Request Body**:
```json
{
  "email": "user@example.com"
}
```

**Response**: `200 OK`
```json
{
  "challenge": "base64url-encoded-challenge",
  "rpId": "example.com",
  "allowCredentials": [
    {
      "id": "base64url-credential-id",
      "type": "public-key"
    }
  ],
  "userVerification": "preferred",
  "timeout": 60000
}
```

**Errors**:
| Code | HTTP | Description |
|------|------|-------------|
| `0000009` | 404 | User not found |
| `0000011` | 400 | No passkeys registered for user |

---

#### POST /v1/authenticate/passkeys:complete

Completes WebAuthn authentication with assertion.

**Request Body**:
```json
{
  "credentialId": "base64url-credential-id",
  "authenticatorData": "base64url-encoded",
  "clientDataJSON": "base64url-encoded",
  "signature": "base64url-encoded",
  "userHandle": "base64url-encoded"
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

**Errors**:
| Code | HTTP | Description |
|------|------|-------------|
| `0000002` | 404 | Challenge not found |
| `0000010` | 404 | Passkey not found |
| `0000012` | 401 | Assertion verification failed |
| `0000013` | 401 | Sign count mismatch (possible cloned authenticator) |
| `0000014` | 400 | Invalid challenge type |

---

### Token Refresh

#### POST /v1/auth/refresh

Refreshes access token using refresh token.

**Request Body**:
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

**Errors**:
| Code | HTTP | Description |
|------|------|-------------|
| `0000015` | 401 | Token invalid or expired |
| `0000016` | 401 | Refresh token revoked |
| `0000017` | 401 | Refresh token not found |

---

## Internal Endpoints

These endpoints are protected by mTLS when `internal-api.mtls-enabled=true`.

### Policy API

#### GET /v1/internal/policies

Returns all active API authorization policies.

**Security**: mTLS client certificate required
- Certificate CN must be in `allowed-principals`
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
  },
  {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "httpMethod": "POST",
    "pathPattern": "/api/v1/wallets/{id}/transactions",
    "permissionCode": "transactions:create",
    "conditionExpr": "@userId == authentication.userId",
    "serviceName": "wallet-service",
    "priority": 20,
    "active": true
  }
]
```

**Query Logic**:
```sql
SELECT ap.id, ap.http_method, ap.path_pattern, p.code,
       ap.condition_expr, ap.service_name, ap.priority
FROM api_permissions ap
INNER JOIN permissions p ON p.id = ap.permission_id
WHERE ap.is_active = true AND p.is_active = true
ORDER BY ap.priority DESC, ap.path_pattern ASC, ap.http_method ASC
```

---

#### GET /v1/internal/policies/version

Returns current policy cache version for invalidation.

**Security**: Same as `/policies`

**Response**: `200 OK`
```json
42
```

**Usage**: Gateway polls this endpoint and reloads cache when version changes.

---

## Error Response Format

All errors follow this format:

```json
{
  "errorCode": "0000001",
  "errorMessage": "User with email was already existed.",
  "timestamp": "2024-01-15T10:30:00Z",
  "path": "/v1/register/passkeys:start",
  "requestId": "abc-123-def"
}
```

---

## Data Types

### Base64URL Encoding

All binary data (challenges, credential IDs, keys) use Base64URL encoding without padding, per WebAuthn spec.

### Timestamps

All timestamps use ISO 8601 format: `2024-01-15T10:30:00Z`

### UUIDs

All IDs are UUID v4 format: `550e8400-e29b-41d4-a716-446655440000`

---

## Rate Limiting

| Endpoint | Limit | Window |
|----------|-------|--------|
| `/v1/register/*` | 10 | 1 minute |
| `/v1/authenticate/*` | 20 | 1 minute |
| `/v1/auth/refresh` | 30 | 1 minute |
| `/v1/internal/*` | 100 | 1 minute |

*Note: Rate limiting is enforced at gateway level, not in this service.*
