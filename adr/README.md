# Architecture Decision Records

This index references ADRs from all Mangala services. ADRs remain in their respective service repositories as the source of truth.

## What is an ADR?

An Architecture Decision Record captures an important architectural decision made along with its context and consequences.

## ADR Index

### Authentication Service

| ADR | Status | Summary |
|-----|--------|---------|
| [ADR-0001: Policy Loading Via Auth Internal API](https://github.com/mangala/mangala-authentication/blob/master/docs/adr/0001-policy-loading-via-auth-internal-api.md) | Accepted | Gateway loads authorization policies from auth service via HTTP API instead of direct database access |
| [ADR-0002: Internal Policy mTLS Authorization](https://github.com/mangala/mangala-authentication/blob/master/docs/adr/0002-internal-policy-mtls-authorization.md) | Accepted | Internal policy endpoints protected by optional mTLS with x509 client certificate authentication |

### Gateway Service

*(No ADRs yet)*

### Cross-Service

*(No cross-service ADRs yet)*

## Local References

If accessing locally (repos as siblings):

- [ADR-0001](../mangala-authentication/docs/adr/0001-policy-loading-via-auth-internal-api.md)
- [ADR-0002](../mangala-authentication/docs/adr/0002-internal-policy-mtls-authorization.md)

## Creating New ADRs

1. Use the [ADR template](./template.md)
2. Number sequentially: `NNNN-short-title.md`
3. Place in the service's `docs/adr/` directory
4. Update this index

## ADR Statuses

| Status | Description |
|--------|-------------|
| Proposed | Under discussion |
| Accepted | Decision made and active |
| Deprecated | No longer applies |
| Superseded | Replaced by another ADR |
