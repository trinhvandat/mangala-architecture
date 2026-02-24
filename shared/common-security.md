# mangala-common-security

Shared security library used by multiple Mangala services (authentication, gateway).

**Package**: `org.mangala.security`
**Location**: `mangala-common-security/src/main/java/org/mangala/security/`

## Overview

This library provides common security models, utilities, and constants shared across services. It ensures consistent handling of:
- JWT claims extraction and representation
- Authorization policy evaluation
- Path and permission matching
- Security-related headers and constants

## Classes

### Models (`model/` package)

#### ApiPermissionDTO

DTO for API permission data transferred from database to gateway.

```java
public class ApiPermissionDTO {
    private String id;           // Policy rule UUID
    private String httpMethod;   // HTTP method (GET, POST, etc.)
    private String pathPattern;  // URL pattern (e.g., /api/v1/wallets/{id})
    private String permissionCode; // Required permission (e.g., wallets:read)
    private String conditionExpr;  // Optional SpEL ABAC condition
    private String serviceName;    // Target service name
    private int priority;          // Rule priority (higher = first)
    private boolean active;        // Is rule active
}
```

**Used by**: Authentication service (to serve policies), Gateway (to enforce policies)

---

#### JwtUserClaims

Represents user claims extracted from JWT token.

```java
public class JwtUserClaims {
    private String userId;              // User UUID (from 'sub' claim)
    private String email;               // User email
    private Set<String> roles;          // Role codes (e.g., ROLE_USER)
    private Set<String> permissions;    // Permission codes (e.g., wallets:read)
    private Map<String, Object> attributes; // Custom attributes for ABAC
    private long issuedAt;              // Token issue timestamp
    private long expiration;            // Token expiration timestamp
    private String issuer;              // Token issuer
    private String tokenId;             // JWT ID (jti claim)
}
```

**Used by**: Gateway (after parsing JWT)

---

#### PolicyRule

Represents an API permission policy rule for authorization. Loaded from database and cached in memory at gateway startup.

```java
public class PolicyRule {
    private String id;
    private String httpMethod;
    private String pathPattern;
    private Set<String> requiredPermissions;
    private String conditionExpr;
    private String serviceName;
    private int priority;
    private transient Pattern compiledPattern;  // Compiled regex for path matching

    public String getCacheKey() {
        return httpMethod + ":" + pathPattern;
    }
}
```

**Used by**: Gateway policy cache

---

#### AuthorizationResult

Result of an authorization decision.

```java
public class AuthorizationResult {
    public enum Decision {
        ALLOW,      // Request authorized
        DENY,       // Request denied (has policy, lacks permission)
        NO_POLICY   // No matching policy found
    }

    private Decision decision;
    private String reason;              // Denial reason
    private Set<String> requiredPermissions;
    private String matchedRule;         // Rule ID that matched
    private long evaluationTimeMs;      // Performance metric

    // Factory methods
    public static AuthorizationResult allow(String matchedRule);
    public static AuthorizationResult deny(String reason, Set<String> requiredPermissions);
    public static AuthorizationResult noPolicy(String path);

    public boolean isAllowed();
}
```

**Used by**: Gateway authorization filter

---

#### PolicyUpdateEvent

Event published when policies are updated. Used to broadcast cache invalidation to all gateway instances.

```java
public class PolicyUpdateEvent {
    public enum UpdateType {
        FULL_RELOAD,    // Reload entire policy cache
        INCREMENTAL     // Only update specific paths
    }

    private long version;
    private UpdateType updateType;
    private List<String> affectedPaths;
    private Instant timestamp;
    private String updatedBy;
}
```

**Used by**: Authentication service (publishes), Gateway (subscribes via Kafka)

---

### Utilities (`util/` package)

#### PermissionMatcher

Utility class for matching permissions with wildcard support.

**Permission Matching**:
```
wallets:*      matches wallets:read, wallets:create, etc.
admin:users:*  matches admin:users:read, admin:users:write
*:*            matches everything (dangerous!)
wallets:read   matches only wallets:read (exact)
```

**Methods**:
```java
// Check if user has a specific permission (with wildcard support)
static boolean hasPermission(Set<String> userPermissions, String required);

// Check if user has ANY of the required permissions
static boolean hasAnyPermission(Set<String> userPermissions, Set<String> required);

// Check if user has ALL required permissions
static boolean hasAllPermissions(Set<String> userPermissions, Set<String> required);

// Compile path pattern to regex (handles {param}, *, **)
static Pattern compilePathPattern(String pathPattern);

// Check if path matches compiled pattern
static boolean matchesPath(Pattern pattern, String path);

// Check if HTTP method matches (* = any method)
static boolean matchesMethod(String patternMethod, String actualMethod);
```

**Path Pattern Compilation**:
```
/api/v1/wallets/{id}  -> ^/api/v1/wallets/[^/]+$
/api/v1/users/**      -> ^/api/v1/users/.*$
/api/v1/files/*       -> ^/api/v1/files/[^/]*$
```

---

#### PathNormalizer

Utility class for normalizing and validating request paths. Prevents path traversal attacks.

**Methods**:
```java
// Normalize path (decode URL, remove //, resolve .., lowercase)
static String normalize(String path);

// Check for traversal attempts (.., %2e%2e, etc.)
static boolean containsTraversal(String path);

// Validate path is safe (starts with /, no traversal, no null bytes)
static boolean isValidPath(String path);

// Extract path variables from pattern
// pattern="/api/v1/wallets/{walletId}", path="/api/v1/wallets/123"
// returns: {walletId: "123"}
static Map<String, String> extractPathVariables(String pattern, String path);
```

---

### Constants (`SecurityConstants.java`)

Shared constants used across security components.

#### HTTP Headers
```java
HEADER_USER_ID = "X-User-Id"
HEADER_USER_EMAIL = "X-User-Email"
HEADER_USER_ROLES = "X-User-Roles"
HEADER_USER_PERMISSIONS = "X-User-Permissions"
HEADER_TENANT_ID = "X-Tenant-Id"
HEADER_REQUEST_ID = "X-Request-Id"
HEADER_CORRELATION_ID = "X-Correlation-Id"
HEADER_AUTHORIZATION = "Authorization"
HEADER_FORBIDDEN_REASON = "X-Forbidden-Reason"
```

#### JWT Claims
```java
CLAIM_ROLES = "roles"
CLAIM_PERMISSIONS = "permissions"
CLAIM_EMAIL = "email"
CLAIM_ATTRIBUTES = "attributes"
CLAIM_TOKEN_TYPE = "type"
CLAIM_TENANT_ID = "tenant_id"
```

#### Token Types
```java
TOKEN_TYPE_ACCESS = "access"
TOKEN_TYPE_REFRESH = "refresh"
BEARER_PREFIX = "Bearer "
```

#### Cache Configuration
```java
REDIS_TOKEN_BLACKLIST_PREFIX = "token:blacklist:"
REDIS_POLICY_CACHE_PREFIX = "policy:cache:"
REDIS_POLICY_VERSION_KEY = "policy:version"

DEFAULT_POLICY_CACHE_SIZE = 10000
DEFAULT_POLICY_CACHE_TTL_SECONDS = 300
DEFAULT_TOKEN_CACHE_TTL_SECONDS = 30
```

#### Kafka Topics
```java
TOPIC_POLICY_UPDATES = "policy-updates"
TOPIC_AUTHORIZATION_AUDIT = "audit.authorization"
```

## Usage Patterns

### Gateway: Policy Evaluation

```java
// 1. Load policies from auth service
List<ApiPermissionDTO> policies = authClient.getPolicies();

// 2. Convert to PolicyRule with compiled patterns
List<PolicyRule> rules = policies.stream()
    .map(dto -> PolicyRule.builder()
        .id(dto.getId())
        .httpMethod(dto.getHttpMethod())
        .pathPattern(dto.getPathPattern())
        .requiredPermissions(Set.of(dto.getPermissionCode()))
        .conditionExpr(dto.getConditionExpr())
        .priority(dto.getPriority())
        .compiledPattern(PermissionMatcher.compilePathPattern(dto.getPathPattern()))
        .build())
    .toList();

// 3. Evaluate request
String normalizedPath = PathNormalizer.normalize(request.getPath());
for (PolicyRule rule : rules) {
    if (PermissionMatcher.matchesMethod(rule.getHttpMethod(), request.getMethod()) &&
        PermissionMatcher.matchesPath(rule.getCompiledPattern(), normalizedPath)) {

        if (PermissionMatcher.hasAnyPermission(userClaims.getPermissions(),
                                                rule.getRequiredPermissions())) {
            return AuthorizationResult.allow(rule.getId());
        } else {
            return AuthorizationResult.deny("Insufficient permissions",
                                            rule.getRequiredPermissions());
        }
    }
}
return AuthorizationResult.noPolicy(normalizedPath);
```

### Gateway: JWT Claims Extraction

```java
// After JWT validation, populate JwtUserClaims
JwtUserClaims claims = JwtUserClaims.builder()
    .userId(jwt.getSubject())
    .email(jwt.getClaim(SecurityConstants.CLAIM_EMAIL))
    .roles(new HashSet<>(jwt.getClaim(SecurityConstants.CLAIM_ROLES)))
    .permissions(new HashSet<>(jwt.getClaim(SecurityConstants.CLAIM_PERMISSIONS)))
    .issuedAt(jwt.getIssuedAt().toEpochMilli())
    .expiration(jwt.getExpiration().toEpochMilli())
    .issuer(jwt.getIssuer())
    .tokenId(jwt.getId())
    .build();

// Forward user context to downstream services via headers
request.header(SecurityConstants.HEADER_USER_ID, claims.getUserId());
request.header(SecurityConstants.HEADER_USER_EMAIL, claims.getEmail());
request.header(SecurityConstants.HEADER_USER_ROLES, String.join(",", claims.getRoles()));
```

## Version Compatibility

When modifying this library:
1. Changes are breaking for all consumers (auth, gateway)
2. Deploy auth service first, then gateway
3. Add new fields as optional with defaults
4. Deprecate before removing fields
