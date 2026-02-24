# Error Code Registry

Cross-service error code registry to prevent collisions and ensure consistent error handling.

## Format

Error codes follow the format: `{SERVICE_PREFIX}{DOMAIN}{SEQUENCE}`

| Prefix | Service | Range |
|--------|---------|-------|
| `00` | mangala-authentication | 0000001 - 0099999 |
| `01` | mangala-gateway | 0100001 - 0199999 |
| `02` | mangala-wallet | 0200001 - 0299999 |

## Authentication Service (00xxxxx)

**Source**: `mangala-authentication/src/main/java/org/mangala/authentication/shared/exception/ErrorConstant.java`

### User Errors

| Code | HTTP Status | Message | Exception |
|------|-------------|---------|-----------|
| `0000001` | 409 Conflict | User with email was already existed | `UserWithEmailExistedException` |
| `0000009` | 404 Not Found | User not found | `UserNotFoundException` |

### Challenge Errors

| Code | HTTP Status | Message | Exception |
|------|-------------|---------|-----------|
| `0000002` | 404 Not Found | Challenge not found for the given session | `ChallengeNotFoundException` |
| `0000003` | 400 Bad Request | Challenge has expired | `ChallengeExpiredException` |
| `0000004` | 400 Bad Request | Challenge has already been used | `ChallengeAlreadyUsedException` |

### Credential Errors

| Code | HTTP Status | Message | Exception |
|------|-------------|---------|-----------|
| `0000005` | 400 Bad Request | Credential verification failed | `CredentialVerificationFailedException` |
| `0000006` | 409 Conflict | Credential ID already exists | `DuplicateCredentialException` |
| `0000007` | 400 Bad Request | Origin validation failed | - |
| `0000008` | 400 Bad Request | Attestation verification failed | - |

### Passkey Errors

| Code | HTTP Status | Message | Exception |
|------|-------------|---------|-----------|
| `0000010` | 404 Not Found | Passkey not found | `PasskeyNotFoundException` |
| `0000011` | 400 Bad Request | No passkeys registered for this user | `NoPasskeysFoundException` |
| `0000012` | 401 Unauthorized | Authentication assertion verification failed | `AssertionVerificationFailedException` |
| `0000013` | 401 Unauthorized | Sign count mismatch detected. Possible cloned authenticator | `SignCountMismatchException` |
| `0000014` | 400 Bad Request | Invalid challenge type for operation | `InvalidChallengeTypeException` |

### Token Errors

| Code | HTTP Status | Message | Exception |
|------|-------------|---------|-----------|
| `0000015` | 401 Unauthorized | Token is invalid or expired | `InvalidTokenException` |
| `0000016` | 401 Unauthorized | Refresh token has been revoked | `RefreshTokenRevokedException` |
| `0000017` | 401 Unauthorized | Refresh token not found | `RefreshTokenNotFoundException` |

## Gateway Service (01xxxxx)

*Reserved range: 0100001 - 0199999*

| Code | HTTP Status | Message | Description |
|------|-------------|---------|-------------|
| `0100001` | 401 Unauthorized | Missing authorization header | No Bearer token provided |
| `0100002` | 401 Unauthorized | Invalid token format | Token is not a valid JWT |
| `0100003` | 401 Unauthorized | Token signature invalid | JWT signature verification failed |
| `0100004` | 401 Unauthorized | Token expired | JWT has expired |
| `0100005` | 403 Forbidden | Insufficient permissions | User lacks required permission |
| `0100006` | 403 Forbidden | ABAC condition failed | SpEL condition evaluated to false |
| `0100007` | 404 Not Found | No policy found | No authorization rule for path |

## Wallet Service (02xxxxx)

*Reserved range: 0200001 - 0299999*

(To be defined when wallet service is implemented)

## Adding New Error Codes

When adding a new error code:

1. Use the next available sequence number in your service's range
2. Update this registry
3. Follow the HTTP status code conventions:
   - `400 Bad Request` - Client error, invalid input
   - `401 Unauthorized` - Authentication required or failed
   - `403 Forbidden` - Authenticated but not authorized
   - `404 Not Found` - Resource doesn't exist
   - `409 Conflict` - Resource conflict (duplicate)
   - `500 Internal Server Error` - Unexpected server error

## Error Response Format

All services should return errors in this format:

```json
{
  "errorCode": "0000001",
  "errorMessage": "User with email was already existed.",
  "timestamp": "2024-01-15T10:30:00Z",
  "path": "/v1/register/passkeys:start",
  "requestId": "abc-123-def"
}
```
