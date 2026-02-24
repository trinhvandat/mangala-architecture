# mangala-authentication Design

## Architecture Overview

The service follows a **feature-first layered architecture** with clean separation of concerns.

### Layer Responsibilities

```
┌─────────────────────────────────────────────────────────────┐
│                    Adapter Layer (Web)                       │
│  Controllers, DTOs, Request/Response mapping                 │
│  Handles HTTP concerns, validation, serialization            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Use Case Layer                            │
│  Application logic, orchestration                            │
│  Interface + Implementation pattern                          │
│  Command objects for input, Response objects for output      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Domain Layer                              │
│  Entities, business exceptions                               │
│  Core business rules                                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                Adapter Layer (Repository)                    │
│  Spring Data JPA repositories                                │
│  Database access                                             │
└─────────────────────────────────────────────────────────────┘
```

### Dependency Rules

- Web adapters depend on Use Cases (interfaces only)
- Use Cases depend on Domain and Repository adapters
- Domain has no external dependencies
- Repository adapters depend on Domain entities

---

## WebAuthn Registration Flow

### Sequence Diagram

```
┌────────┐          ┌─────────────┐          ┌────────────────┐          ┌────────┐
│ Client │          │ AuthControl │          │ Registration   │          │   DB   │
│        │          │     ler     │          │   UseCases     │          │        │
└───┬────┘          └──────┬──────┘          └───────┬────────┘          └───┬────┘
    │                      │                         │                       │
    │ POST /register/      │                         │                       │
    │ passkeys:start       │                         │                       │
    │ {email}              │                         │                       │
    │─────────────────────>│                         │                       │
    │                      │ startRegistration()     │                       │
    │                      │────────────────────────>│                       │
    │                      │                         │ Check email exists    │
    │                      │                         │──────────────────────>│
    │                      │                         │<──────────────────────│
    │                      │                         │                       │
    │                      │                         │ Create user           │
    │                      │                         │──────────────────────>│
    │                      │                         │<──────────────────────│
    │                      │                         │                       │
    │                      │                         │ Generate challenge    │
    │                      │                         │ (32 bytes random)     │
    │                      │                         │                       │
    │                      │                         │ Save challenge        │
    │                      │                         │──────────────────────>│
    │                      │                         │<──────────────────────│
    │                      │<────────────────────────│                       │
    │<─────────────────────│                         │                       │
    │ {sessionId,          │                         │                       │
    │  challenge,          │                         │                       │
    │  rp, user,           │                         │                       │
    │  pubKeyCredParams}   │                         │                       │
    │                      │                         │                       │
    │ [Browser creates     │                         │                       │
    │  credential via      │                         │                       │
    │  WebAuthn API]       │                         │                       │
    │                      │                         │                       │
    │ POST /register/      │                         │                       │
    │ passkeys:complete    │                         │                       │
    │ {sessionId,          │                         │                       │
    │  credential}         │                         │                       │
    │─────────────────────>│                         │                       │
    │                      │ completeRegistration()  │                       │
    │                      │────────────────────────>│                       │
    │                      │                         │ Get challenge         │
    │                      │                         │──────────────────────>│
    │                      │                         │<──────────────────────│
    │                      │                         │                       │
    │                      │                         │ Verify attestation    │
    │                      │                         │ (WebAuthn4J)          │
    │                      │                         │                       │
    │                      │                         │ Check duplicate       │
    │                      │                         │──────────────────────>│
    │                      │                         │<──────────────────────│
    │                      │                         │                       │
    │                      │                         │ Save passkey          │
    │                      │                         │──────────────────────>│
    │                      │                         │<──────────────────────│
    │                      │                         │                       │
    │                      │                         │ Mark challenge used   │
    │                      │                         │──────────────────────>│
    │                      │<────────────────────────│                       │
    │<─────────────────────│                         │                       │
    │ {userId,             │                         │                       │
    │  credentialId}       │                         │                       │
    │                      │                         │                       │
```

### Key Components

**StartRegistrationUseCaseImpl**:
1. Validates email uniqueness (if provided)
2. Creates user record
3. Generates 32-byte cryptographic challenge
4. Stores challenge with session ID, expiration, IP, User-Agent
5. Returns WebAuthn registration options

**CompleteRegistrationUseCaseImpl**:
1. Retrieves challenge by session ID
2. Validates challenge not expired, not used
3. Verifies attestation using WebAuthn4J
4. Checks credential ID not already registered
5. Extracts device info from User-Agent
6. Saves passkey with public key, algorithm, transports
7. Marks challenge as used

---

## WebAuthn Authentication Flow

### Sequence Diagram

```
┌────────┐          ┌─────────────┐          ┌────────────────┐          ┌────────┐
│ Client │          │ AuthControl │          │ Authentication │          │   DB   │
│        │          │     ler     │          │   UseCases     │          │        │
└───┬────┘          └──────┬──────┘          └───────┬────────┘          └───┬────┘
    │                      │                         │                       │
    │ POST /authenticate/  │                         │                       │
    │ passkeys:start       │                         │                       │
    │ {email}              │                         │                       │
    │─────────────────────>│                         │                       │
    │                      │ startAuthentication()   │                       │
    │                      │────────────────────────>│                       │
    │                      │                         │ Find user by email    │
    │                      │                         │──────────────────────>│
    │                      │                         │<──────────────────────│
    │                      │                         │                       │
    │                      │                         │ Get user passkeys     │
    │                      │                         │──────────────────────>│
    │                      │                         │<──────────────────────│
    │                      │                         │                       │
    │                      │                         │ Generate challenge    │
    │                      │                         │ Save challenge        │
    │                      │                         │──────────────────────>│
    │                      │<────────────────────────│                       │
    │<─────────────────────│                         │                       │
    │ {challenge,          │                         │                       │
    │  allowCredentials,   │                         │                       │
    │  rpId}               │                         │                       │
    │                      │                         │                       │
    │ [Browser gets        │                         │                       │
    │  assertion via       │                         │                       │
    │  WebAuthn API]       │                         │                       │
    │                      │                         │                       │
    │ POST /authenticate/  │                         │                       │
    │ passkeys:complete    │                         │                       │
    │ {credentialId,       │                         │                       │
    │  authenticatorData,  │                         │                       │
    │  clientDataJSON,     │                         │                       │
    │  signature}          │                         │                       │
    │─────────────────────>│                         │                       │
    │                      │ completeAuthentication()│                       │
    │                      │────────────────────────>│                       │
    │                      │                         │ Get passkey           │
    │                      │                         │──────────────────────>│
    │                      │                         │<──────────────────────│
    │                      │                         │                       │
    │                      │                         │ Get challenge         │
    │                      │                         │──────────────────────>│
    │                      │                         │<──────────────────────│
    │                      │                         │                       │
    │                      │                         │ Verify assertion      │
    │                      │                         │ (WebAuthn4J)          │
    │                      │                         │                       │
    │                      │                         │ Validate sign count   │
    │                      │                         │                       │
    │                      │                         │ Update passkey        │
    │                      │                         │──────────────────────>│
    │                      │                         │                       │
    │                      │                         │ Get user roles/perms  │
    │                      │                         │──────────────────────>│
    │                      │                         │<──────────────────────│
    │                      │                         │                       │
    │                      │                         │ Generate JWT tokens   │
    │                      │                         │                       │
    │                      │                         │ Save refresh token    │
    │                      │                         │──────────────────────>│
    │                      │<────────────────────────│                       │
    │<─────────────────────│                         │                       │
    │ {accessToken,        │                         │                       │
    │  refreshToken,       │                         │                       │
    │  expiresIn}          │                         │                       │
```

### Sign Count Validation

The service tracks `signCount` to detect cloned authenticators:

1. Each assertion includes a sign count from the authenticator
2. Server compares with stored sign count
3. If new count <= stored count, possible clone detected
4. Throws `SignCountMismatchException` if validation fails
5. Updates stored count on successful authentication

---

## JWT Token Lifecycle

### Token Generation

```
┌──────────────────────────────────────────────────────────────┐
│                    JwtService.generateTokens()                │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Access Token (short-lived)                                  │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Header: {alg: HS256, typ: JWT}                         │  │
│  │ Payload:                                               │  │
│  │   sub: user UUID                                       │  │
│  │   iss: "mangala"                                       │  │
│  │   aud: "mangala-gateway"                               │  │
│  │   iat: issued timestamp                                │  │
│  │   exp: iat + 900 (15 min)                              │  │
│  │   jti: unique token ID                                 │  │
│  │   type: "access"                                       │  │
│  │   email: user email                                    │  │
│  │   roles: ["ROLE_USER"]                                 │  │
│  │   permissions: ["wallets:read", ...]                   │  │
│  │   amr: ["passkey"]                                     │  │
│  │ Signature: HMAC-SHA256(secret)                         │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  Refresh Token (long-lived)                                  │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Payload:                                               │  │
│  │   sub: user UUID                                       │  │
│  │   iss: "mangala"                                       │  │
│  │   aud: "mangala-auth-refresh"                          │  │
│  │   iat: issued timestamp                                │  │
│  │   exp: iat + 604800 (7 days)                           │  │
│  │   jti: unique token ID (stored in DB)                  │  │
│  │   type: "refresh"                                      │  │
│  │   email: user email                                    │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Token Refresh Flow

```
┌────────┐          ┌─────────────┐          ┌────────────────┐
│ Client │          │ AuthControl │          │ RefreshToken   │
│        │          │     ler     │          │   UseCase      │
└───┬────┘          └──────┬──────┘          └───────┬────────┘
    │                      │                         │
    │ POST /auth/refresh   │                         │
    │ {refreshToken}       │                         │
    │─────────────────────>│                         │
    │                      │ refreshToken()          │
    │                      │────────────────────────>│
    │                      │                         │
    │                      │                         │ 1. Parse JWT
    │                      │                         │ 2. Validate signature
    │                      │                         │ 3. Check type="refresh"
    │                      │                         │ 4. Check audience
    │                      │                         │ 5. Look up token in DB
    │                      │                         │ 6. Check not revoked
    │                      │                         │ 7. Check not expired
    │                      │                         │ 8. Fetch user roles/perms
    │                      │                         │ 9. Generate new tokens
    │                      │                         │ 10. Revoke old refresh
    │                      │                         │ 11. Save new refresh
    │                      │                         │
    │                      │<────────────────────────│
    │<─────────────────────│                         │
    │ {accessToken,        │                         │
    │  refreshToken (new), │                         │
    │  expiresIn}          │                         │
```

**Key behaviors**:
- Refresh tokens are **single-use** (old token revoked)
- Roles/permissions are **fetched fresh** (not from old token)
- New refresh token is **stored in database**

---

## Data Model

### Entity Relationship Diagram

```
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│     users       │       │   user_roles    │       │     roles       │
├─────────────────┤       ├─────────────────┤       ├─────────────────┤
│ id (PK)         │──┐    │ id (PK)         │    ┌──│ id (PK)         │
│ email           │  │    │ user_id (FK)    │────┘  │ code            │
│ email_verified  │  └───>│ role_id (FK)    │───────│ name            │
│ created_at      │       └─────────────────┘       │ is_active       │
│ created_by      │                                 └────────┬────────┘
└────────┬────────┘                                          │
         │                                                   │
         │         ┌─────────────────┐       ┌───────────────┴───────┐
         │         │ role_permissions│       │    permissions        │
         │         ├─────────────────┤       ├───────────────────────┤
         │         │ id (PK)         │       │ id (PK)               │
         │         │ role_id (FK)    │───────│ code                  │
         │         │ permission_id   │──────>│ name                  │
         │         └─────────────────┘       │ category              │
         │                                   │ is_active             │
         │                                   └───────────┬───────────┘
         │                                               │
         │                                   ┌───────────┴───────────┐
         │                                   │   api_permissions     │
         │                                   ├───────────────────────┤
         │                                   │ id (PK)               │
         │                                   │ http_method           │
         │                                   │ path_pattern          │
         │                                   │ permission_id (FK)    │
         │                                   │ condition_expr        │
         │                                   │ service_name          │
         │                                   │ priority              │
         │                                   │ is_active             │
         │                                   └───────────────────────┘
         │
         │         ┌─────────────────────┐
         └────────>│   user_passkeys     │
                   ├─────────────────────┤
                   │ id (PK)             │
                   │ user_id (FK)        │
                   │ credential_id       │
                   │ public_key          │
                   │ algorithm           │
                   │ sign_count          │
                   │ transports          │
                   │ device_name         │
                   │ is_active           │
                   └─────────────────────┘

┌─────────────────────┐       ┌─────────────────────┐
│ passkey_challenges  │       │   refresh_tokens    │
├─────────────────────┤       ├─────────────────────┤
│ id (PK)             │       │ id (PK)             │
│ challenge           │       │ token_id            │
│ user_id (FK)        │       │ user_id (FK)        │
│ session_id          │       │ expires_at          │
│ operation_type      │       │ revoked_at          │
│ expires_at          │       │ created_at          │
│ is_used             │       └─────────────────────┘
└─────────────────────┘

┌─────────────────────┐
│   policy_version    │
├─────────────────────┤
│ id (always 1)       │
│ version             │
│ updated_at          │
└─────────────────────┘
```

---

## Security Model

### Internal API Protection (mTLS)

```
┌─────────────────┐                      ┌─────────────────────────┐
│    Gateway      │                      │    Authentication       │
│                 │                      │                         │
│ ┌─────────────┐ │    mTLS Handshake    │ ┌─────────────────────┐ │
│ │ Client Cert │─┼──────────────────────┼─│InternalApiSecurity  │ │
│ │ CN=gateway  │ │                      │ │     Config          │ │
│ └─────────────┘ │                      │ │                     │ │
│                 │                      │ │ 1. Extract CN from  │ │
│                 │                      │ │    certificate      │ │
│                 │                      │ │ 2. Check allowlist  │ │
│                 │                      │ │ 3. Grant authority  │ │
│                 │                      │ └─────────────────────┘ │
└─────────────────┘                      └─────────────────────────┘
```

**Configuration**:
```yaml
application.security.internal-api:
  mtls-enabled: true
  subject-principal-regex: "CN=(.*?)(?:,|$)"
  allowed-principals: "gateway-service"
```

**Behavior**:
- When `mtls-enabled=true`: Requires valid x509 client cert
- Extracts CN using regex
- Grants `INTERNAL_POLICY_READ` authority if CN in allowlist
- Returns 403 if not authorized

### WebAuthn Security

- Challenges expire after configurable timeout
- Challenges are single-use (marked as used after verification)
- Sign count validation detects cloned authenticators
- Attestation verification (configurable: none/direct/indirect)
- Origin validation against configured RP origin
