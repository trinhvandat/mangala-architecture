# mangala-wallet-service API

## Base URL

```
http://localhost:8081/api/v1
```

Via Gateway:
```
http://localhost:8000/api/v1
```

## Authentication

All endpoints require JWT authentication via Bearer token:

```http
Authorization: Bearer <access_token>
```

---

## Wallet Endpoints

### Create Wallet

```http
POST /wallets
```

Creates a new wallet for the authenticated user.

**Request Body:**
```json
{
  "address": "0x742d35Cc6634C0532925a3b844Bc9e7595f8B2c1",
  "chainType": "ETHEREUM",
  "label": "My Main Wallet"
}
```

**Response:** `201 Created`
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "address": "0x742d35Cc6634C0532925a3b844Bc9e7595f8B2c1",
  "chainType": "ETHEREUM",
  "label": "My Main Wallet",
  "isActive": true,
  "createdAt": "2025-02-25T10:30:00Z",
  "lastSyncedAt": null
}
```

**Errors:**
| Code | Error | Description |
|------|-------|-------------|
| 400 | WAL-002 | Wallet already exists for this address and chain |
| 400 | - | Invalid address format |

---

### Get Wallet

```http
GET /wallets/{walletId}
```

**Response:** `200 OK`
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "address": "0x742d35Cc6634C0532925a3b844Bc9e7595f8B2c1",
  "chainType": "ETHEREUM",
  "label": "My Main Wallet",
  "isActive": true,
  "createdAt": "2025-02-25T10:30:00Z",
  "lastSyncedAt": "2025-02-25T10:35:00Z"
}
```

**Errors:**
| Code | Error | Description |
|------|-------|-------------|
| 404 | WAL-001 | Wallet not found |
| 403 | WAL-003 | Access denied to this wallet |

---

### List User Wallets

```http
GET /wallets
```

Returns all active wallets for the authenticated user.

**Response:** `200 OK`
```json
[
  {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "address": "0x742d35Cc6634C0532925a3b844Bc9e7595f8B2c1",
    "chainType": "ETHEREUM",
    "label": "My Main Wallet",
    "isActive": true,
    "createdAt": "2025-02-25T10:30:00Z",
    "lastSyncedAt": "2025-02-25T10:35:00Z"
  }
]
```

---

### Delete Wallet

```http
DELETE /wallets/{walletId}
```

Soft-deletes a wallet (sets `isActive=false`).

**Response:** `204 No Content`

**Errors:**
| Code | Error | Description |
|------|-------|-------------|
| 404 | WAL-001 | Wallet not found |
| 403 | WAL-003 | Access denied to this wallet |

---

## Balance Endpoints

### Get Wallet Balances

```http
GET /wallets/{walletId}/balances
```

Returns all token balances for a wallet.

**Response:** `200 OK`
```json
{
  "walletId": "550e8400-e29b-41d4-a716-446655440000",
  "chain": "ETHEREUM",
  "address": "0x742d35Cc6634C0532925a3b844Bc9e7595f8B2c1",
  "balances": [
    {
      "tokenAddress": null,
      "symbol": "ETH",
      "name": "Ethereum",
      "decimals": 18,
      "balance": "1.234567890123456789",
      "balanceRaw": "1234567890123456789",
      "lastSyncedAt": "2025-02-25T10:35:00Z"
    },
    {
      "tokenAddress": "0xdAC17F958D2ee523a2206206994597C13D831ec7",
      "symbol": "USDT",
      "name": "Tether USD",
      "decimals": 6,
      "balance": "1000.000000",
      "balanceRaw": "1000000000",
      "lastSyncedAt": "2025-02-25T10:35:00Z"
    }
  ],
  "totalTokens": 2,
  "lastSyncedAt": "2025-02-25T10:35:00Z"
}
```

**Notes:**
- `tokenAddress: null` indicates native token (ETH, BNB, etc.)
- `balance` is human-readable (divided by 10^decimals)
- `balanceRaw` is the raw blockchain value as string

**Errors:**
| Code | Error | Description |
|------|-------|-------------|
| 404 | WAL-001 | Wallet not found |
| 403 | WAL-003 | Access denied to this wallet |

---

### Sync Wallet Balances

```http
POST /wallets/{walletId}/sync
```

Triggers an immediate balance sync for a wallet.

**Response:** `200 OK`
```json
{
  "walletId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "COMPLETED",
  "tokensFound": 5,
  "syncDurationMs": 1234,
  "message": "Balance sync completed successfully"
}
```

**Possible Status Values:**
| Status | Description |
|--------|-------------|
| COMPLETED | Sync finished successfully |
| FAILED | Sync failed (see message for details) |
| PARTIAL | Some tokens failed but native succeeded |

**Errors:**
| Code | Error | Description |
|------|-------|-------------|
| 404 | WAL-001 | Wallet not found |
| 403 | WAL-003 | Access denied to this wallet |
| 503 | - | Chain RPC unavailable |

---

## Kafka Events

### balance.updates

Published after successful balance sync.

**Topic:** `balance.updates`

**Key:** `{walletId}` (for partition ordering)

**Schema:**
```json
{
  "eventId": "550e8400-e29b-41d4-a716-446655440000-1708858200000",
  "eventType": "balance.updates",
  "walletId": "550e8400-e29b-41d4-a716-446655440000",
  "chain": "ethereum",
  "address": "0x742d35Cc6634C0532925a3b844Bc9e7595f8B2c1",
  "balances": [
    {
      "symbol": "ETH",
      "contractAddress": null,
      "amount": "1.234567890123456789",
      "decimals": 18
    },
    {
      "symbol": "USDT",
      "contractAddress": "0xdAC17F958D2ee523a2206206994597C13D831ec7",
      "amount": "1000.000000",
      "decimals": 6
    }
  ],
  "timestamp": "2025-02-25T10:35:00Z"
}
```

**Consumers:**
- Portfolio Service (aggregates holdings)
- Notification Service (balance alerts)

---

## Error Response Format

All errors follow standard format:

```json
{
  "code": "WAL-001",
  "message": "Wallet not found",
  "timestamp": "2025-02-25T10:30:00Z",
  "path": "/api/v1/wallets/invalid-id"
}
```

### Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| WAL-001 | 404 | Wallet not found |
| WAL-002 | 400 | Wallet already exists for this address and chain |
| WAL-003 | 403 | Access denied to this wallet |

---

## Rate Limits

Via Gateway:
- Default: 10 requests/second, burst 20
- Sync endpoint: 1 request/minute per wallet (recommended)

---

## Supported Chains

| Chain | chainType Value | Native Token |
|-------|-----------------|--------------|
| Ethereum | `ETHEREUM` | ETH |
| BSC | `BSC` | BNB |
| Polygon | `POLYGON` | MATIC |
| Arbitrum | `ARBITRUM` | ETH |
