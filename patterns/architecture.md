# Mangala Architecture Patterns

This document defines the coding patterns and conventions that ALL Mangala services must follow.

## Feature-First Layered Architecture

### Structure

Organize code by **feature first**, then by **layer**:

```
src/main/java/org/mangala/{service}/
├── {feature}/                      # Feature module
│   ├── adapter/
│   │   ├── web/                    # Controllers, DTOs, mappers
│   │   └── repository/             # JPA repositories
│   ├── usecase/                    # Use case interfaces & implementations
│   └── domain/                     # Entities, business exceptions
│
├── shared/                         # Cross-cutting concerns
│   ├── config/                     # Configuration classes
│   └── exception/                  # Error definitions
│
└── MainApplication.java
```

### Example

```
org.mangala.authentication/
├── auth/
│   ├── adapter/web/AuthController.java
│   ├── usecase/RefreshTokenUseCase.java
│   └── domain/RefreshTokenEntity.java
├── passkey/
│   ├── adapter/repository/UserPasskeyRepository.java
│   ├── usecase/StartRegistrationUseCase.java
│   └── domain/UserPasskeyEntity.java
└── user/
    ├── usecase/CreateUserUseCase.java
    └── domain/UserEntity.java
```

---

## Use Case Pattern

### Interface + Implementation

Every use case has an interface and implementation:

```java
// Interface
public interface StartRegistrationUseCase {
    StartRegisterPasskeyResponse execute(StartRegistrationCommand command);
}

// Implementation
@Service
@RequiredArgsConstructor
public class StartRegistrationUseCaseImpl implements StartRegistrationUseCase {

    private final UserRepository userRepository;
    private final ChallengeService challengeService;

    @Override
    public StartRegisterPasskeyResponse execute(StartRegistrationCommand command) {
        // Implementation
    }
}
```

### Command Objects

Use command objects for use case input:

```java
@Data
@Builder
public class StartRegistrationCommand {
    private final String email;
    private final String ipAddress;
    private final String userAgent;
}
```

### Response Objects

Use dedicated response objects for output:

```java
@Data
@Builder
public class StartRegisterPasskeyResponse {
    private final String sessionId;
    private final String challenge;
    private final RelyingPartyInfo rp;
    private final UserInfo user;
}
```

### Benefits

- Clear contracts between layers
- Easy to test (mock interfaces)
- Single Responsibility Principle
- Explicit input/output

---

## DTO Mapping

### MapStruct

Use MapStruct for mapping between layers:

```java
@Mapper(componentModel = "spring")
public interface PasskeyResponseMapper {

    PasskeyRegistrationOptionsDTO toDto(StartRegisterPasskeyResponse response);

    @Mapping(target = "credentialId", expression = "java(encodeBase64Url(response.getCredentialId()))")
    CompletePasskeyRegistrationResponseDTO toDto(CompletePasskeyRegistrationResponse response);

    default String encodeBase64Url(byte[] data) {
        return Base64.getUrlEncoder().withoutPadding().encodeToString(data);
    }
}
```

### Naming Convention

| Layer | Suffix | Example |
|-------|--------|---------|
| Controller input | `RequestDTO` | `CompletePasskeyRegistrationRequestDTO` |
| Controller output | `ResponseDTO` | `PasskeyRegistrationOptionsDTO` |
| Use case input | `Command` | `StartRegistrationCommand` |
| Use case output | `Response` | `StartRegisterPasskeyResponse` |

---

## Repository Pattern

### Spring Data JPA

Use Spring Data interfaces:

```java
public interface UserRepository extends JpaRepository<UserEntity, UUID> {

    Optional<UserEntity> findByEmail(String email);

    boolean existsByEmail(String email);
}
```

### Custom Queries

Use `@Query` for complex queries:

```java
public interface UserAuthorizationQueryRepository {

    @Query("""
        SELECT r.code FROM RoleEntity r
        INNER JOIN UserRoleEntity ur ON ur.roleId = r.id
        WHERE ur.userId = :userId AND r.isActive = true
        """)
    List<String> findRolesByUserId(@Param("userId") UUID userId);
}
```

### Native Queries

Use native SQL only when JPA is insufficient:

```java
@Query(value = """
    SELECT p.code FROM permissions p
    INNER JOIN role_permissions rp ON rp.permission_id = p.id
    INNER JOIN user_roles ur ON ur.role_id = rp.role_id
    WHERE ur.user_id = :userId AND p.is_active = true
    """, nativeQuery = true)
List<String> findPermissionsByUserId(@Param("userId") UUID userId);
```

---

## Exception Handling

### Error Definition Enum

Define all errors in one place:

```java
@AllArgsConstructor
@Getter
public enum ErrorConstant implements ErrorDefinition {
    USER_NOT_FOUND(HttpStatus.NOT_FOUND, "0000009", "User not found."),
    CHALLENGE_EXPIRED(HttpStatus.BAD_REQUEST, "0000003", "Challenge has expired.");

    private final HttpStatus httpStatus;
    private final String errorCode;
    private final String errorMessage;
}
```

### Domain Exceptions

Extend `BaseException` from `mangala-exception`:

```java
public class UserNotFoundException extends BaseException {
    public UserNotFoundException() {
        super(ErrorConstant.USER_NOT_FOUND);
    }
}
```

### Exception Hierarchy

```
BaseException (from mangala-exception)
├── UserNotFoundException
├── ChallengeExpiredException
├── InvalidTokenException
└── ...
```

### HTTP Status Guidelines

| Status | Use When |
|--------|----------|
| 400 Bad Request | Client error, invalid input |
| 401 Unauthorized | Authentication required/failed |
| 403 Forbidden | Authenticated but not authorized |
| 404 Not Found | Resource doesn't exist |
| 409 Conflict | Resource conflict (duplicate) |

---

## Naming Conventions

### Java

| Element | Convention | Example |
|---------|------------|---------|
| Class | PascalCase | `UserPasskeyEntity` |
| Interface | PascalCase | `StartRegistrationUseCase` |
| Method | camelCase | `findByEmail` |
| Field | camelCase | `credentialId` |
| Constant | SCREAMING_SNAKE | `TOKEN_TYPE_ACCESS` |
| Package | lowercase | `org.mangala.authentication` |

### Use Case Naming

| Type | Pattern | Example |
|------|---------|---------|
| Interface | `{Action}{Entity}UseCase` | `StartRegistrationUseCase` |
| Implementation | `{Action}{Entity}UseCaseImpl` | `StartRegistrationUseCaseImpl` |

### DTO Naming

| Type | Pattern | Example |
|------|---------|---------|
| Request | `{Action}{Entity}RequestDTO` | `CompletePasskeyRegistrationRequestDTO` |
| Response | `{Entity}{Action}ResponseDTO` | `PasskeyRegistrationOptionsDTO` |

### Database

| Element | Convention | Example |
|---------|------------|---------|
| Table | snake_case, plural | `user_passkeys` |
| Column | snake_case | `credential_id` |
| Primary key | `id` | `id` |
| Foreign key | `{table}_id` | `user_id` |
| Index | `idx_{table}_{column}` | `idx_users_email` |
| Unique | `uq_{table}_{column}` | `uq_users_email` |

---

## Database Migrations

### Flyway

Use Flyway for versioned migrations.

**File naming**: `V{N}__{description}.sql`

```
src/main/resources/migration/
├── V1__create_users_table.sql
├── V2__create_user_passkeys_table.sql
├── V3__create_passkey_challenges_table.sql
└── V4__add_refresh_tokens_table.sql
```

### Migration Guidelines

1. **Never modify existing migrations** in production
2. **Always add** new migrations for changes
3. **Include rollback comments** (even if not automated)
4. **Test migrations** against production-like data

### Example Migration

```sql
-- V5__create_permissions_tables.sql

CREATE TABLE permissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(100) NOT NULL UNIQUE,
    name VARCHAR(200) NOT NULL,
    description VARCHAR(500),
    category VARCHAR(50),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMP NOT NULL DEFAULT now(),
    created_by UUID
);

CREATE INDEX idx_permissions_code ON permissions(code);
CREATE INDEX idx_permissions_category ON permissions(category);

-- Rollback:
-- DROP INDEX idx_permissions_category;
-- DROP INDEX idx_permissions_code;
-- DROP TABLE permissions;
```

---

## Testing Patterns

### Unit Tests

Test use cases with mocked dependencies:

```java
@ExtendWith(MockitoExtension.class)
class StartRegistrationUseCaseImplTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private StartRegistrationUseCaseImpl useCase;

    @Test
    void execute_shouldCreateUserAndChallenge() {
        // Given
        var command = StartRegistrationCommand.builder()
            .email("test@example.com")
            .build();
        when(userRepository.existsByEmail(any())).thenReturn(false);

        // When
        var result = useCase.execute(command);

        // Then
        assertThat(result.getSessionId()).isNotNull();
        verify(userRepository).save(any());
    }
}
```

### Integration Tests

Use Testcontainers for database tests:

```java
@SpringBootTest
@Testcontainers
class UserRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @Autowired
    private UserRepository userRepository;

    @Test
    void findByEmail_shouldReturnUser() {
        // Test with real database
    }
}
```

### Test Naming

Use descriptive names: `{method}_{scenario}_{expectedResult}`

```java
void execute_whenEmailExists_shouldThrowException()
void findByEmail_whenUserNotFound_shouldReturnEmpty()
void refresh_whenTokenRevoked_shouldReturnUnauthorized()
```

---

## Configuration Patterns

### Properties Classes

Use `@ConfigurationProperties` for type-safe config:

```java
@ConfigurationProperties(prefix = "application.authentication.jwt")
@Data
public class JwtProperties {
    private String secret;
    private String issuer;
    private String accessTokenAudience = "mangala-gateway";
    private String refreshTokenAudience = "mangala-auth-refresh";
    private long accessTokenExpirationSeconds = 900;
    private long refreshTokenExpirationSeconds = 604800;
}
```

### Environment Variables

Map to environment variables:

```yaml
application:
  authentication:
    jwt:
      secret: ${JWT_SECRET}
      issuer: ${JWT_ISSUER:mangala}
      access-token-expiration-seconds: ${JWT_ACCESS_EXPIRATION_SECONDS:900}
```

### Validation

Validate configuration at startup:

```java
@Component
public class ConfigValidationProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean instanceof JwtProperties props) {
            if (props.getSecret() == null || props.getSecret().length() < 32) {
                throw new IllegalStateException("JWT secret must be at least 256 bits");
            }
        }
        return bean;
    }
}
```

---

## API Design

### REST Conventions

| Action | HTTP Method | Path Pattern |
|--------|-------------|--------------|
| Create | POST | `/v1/resources` |
| Read one | GET | `/v1/resources/{id}` |
| Read list | GET | `/v1/resources` |
| Update | PUT | `/v1/resources/{id}` |
| Partial update | PATCH | `/v1/resources/{id}` |
| Delete | DELETE | `/v1/resources/{id}` |
| Custom action | POST | `/v1/resources/{id}:{action}` |

### Custom Actions

Use colon notation for non-CRUD operations:

```
POST /v1/register/passkeys:start
POST /v1/register/passkeys:complete
POST /v1/authenticate/passkeys:start
POST /v1/authenticate/passkeys:complete
```

### Versioning

- Version in URL path: `/v1/`, `/v2/`
- Never break existing versions
- Deprecate before removing

### Error Responses

Always return structured errors:

```json
{
  "errorCode": "0000001",
  "errorMessage": "User with email was already existed.",
  "timestamp": "2024-01-15T10:30:00Z",
  "path": "/v1/register/passkeys:start"
}
```
