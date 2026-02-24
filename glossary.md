# Mangala Domain Glossary

Common terms used across Mangala services.

## Authentication & Identity

| Term | Definition |
|------|------------|
| **Passkey** | A FIDO2/WebAuthn credential that replaces passwords. Stored on user's device (phone, laptop). Uses public-key cryptography. |
| **WebAuthn** | W3C standard for web authentication using public-key cryptography. The browser API for passkeys. |
| **FIDO2** | The overall framework that includes WebAuthn (browser API) and CTAP (device protocol). |
| **Relying Party (RP)** | The service that relies on the authenticator for user verification. In our case, Mangala. |
| **Credential** | A WebAuthn credential consisting of a credential ID and public key, stored on the server. |
| **Authenticator** | The device/software that generates and stores passkeys (e.g., iPhone, YubiKey, Windows Hello). |
| **Challenge** | Random bytes sent by server that client must sign to prove possession of private key. |
| **Attestation** | Proof from the authenticator about its properties during registration. |
| **Assertion** | Signed proof from authenticator during authentication. |
| **Sign Count** | Counter incremented by authenticator on each use. Detects cloned authenticators. |
| **User Verification** | Local verification on device (biometric, PIN) before cryptographic operation. |

## Tokens & Sessions

| Term | Definition |
|------|------------|
| **JWT** | JSON Web Token. Compact, URL-safe token format for claims. |
| **Access Token** | Short-lived JWT used to access protected resources. Contains user identity and permissions. |
| **Refresh Token** | Long-lived token used to obtain new access tokens without re-authentication. |
| **Token Rotation** | Security practice where refresh tokens are single-use; new one issued each refresh. |
| **Claims** | Key-value pairs in a JWT (e.g., `sub`, `roles`, `permissions`). |
| **Bearer Token** | Token sent in `Authorization: Bearer <token>` header. |

## Authorization

| Term | Definition |
|------|------------|
| **RBAC** | Role-Based Access Control. Users have roles, roles have permissions. |
| **ABAC** | Attribute-Based Access Control. Decisions based on attributes (user, resource, environment). |
| **Permission** | A specific action allowed on a resource (e.g., `wallets:read`, `transactions:create`). |
| **Role** | A collection of permissions assigned to users (e.g., `ROLE_USER`, `ROLE_ADMIN`). |
| **Policy** | A rule that maps HTTP method + path pattern to required permission. |
| **Policy Rule** | Single authorization rule evaluated by gateway. |
| **Condition Expression** | SpEL expression for ABAC conditions (e.g., `@userId == resource.ownerId`). |

## Security

| Term | Definition |
|------|------------|
| **mTLS** | Mutual TLS. Both client and server present certificates for authentication. |
| **x509** | Standard format for public key certificates. Used in mTLS. |
| **Certificate** | Digital document binding public key to identity. Contains subject DN, issuer, validity. |
| **CN (Common Name)** | Field in certificate subject identifying the entity (e.g., `gateway-service`). |
| **HMAC** | Hash-based Message Authentication Code. Used for JWT signing. |
| **HS256** | HMAC with SHA-256. JWT signing algorithm. |

## Architecture

| Term | Definition |
|------|------------|
| **Use Case** | Application service that orchestrates a single business operation. |
| **Command** | Input object for a use case containing all required data. |
| **DTO** | Data Transfer Object. Used to transfer data between layers. |
| **Entity** | Domain object with identity, mapped to database table. |
| **Repository** | Abstraction for data access. Interface to persistence. |
| **Adapter** | Implementation that connects domain to external systems (web, database). |

## Infrastructure

| Term | Definition |
|------|------------|
| **Gateway** | API Gateway that handles routing, authentication, and authorization. |
| **Policy Cache** | In-memory cache of authorization policies in gateway for fast evaluation. |
| **Policy Version** | Version number incremented when policies change. Used for cache invalidation. |
| **Flyway** | Database migration tool. Manages schema versioning. |

## Data Formats

| Term | Definition |
|------|------------|
| **Base64URL** | URL-safe Base64 encoding without padding. Used in WebAuthn and JWT. |
| **CBOR** | Concise Binary Object Representation. Used for WebAuthn public keys. |
| **COSE** | CBOR Object Signing and Encryption. Format for WebAuthn keys. |
| **UUID** | Universally Unique Identifier. 128-bit identifier (e.g., `550e8400-e29b-41d4-a716-446655440000`). |

## Algorithms

| Term | Definition |
|------|------------|
| **ES256** | ECDSA with P-256 curve and SHA-256. COSE algorithm -7. |
| **RS256** | RSASSA-PKCS1-v1_5 with SHA-256. COSE algorithm -257. |
| **EdDSA** | Edwards-curve Digital Signature Algorithm. COSE algorithm -8. |
