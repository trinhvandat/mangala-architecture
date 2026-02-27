# mangala-portfolio API

## Base URL

`/api/v1` - All endpoints are versioned under v1.

## Portfolio Management Endpoints

### Create Portfolio

#### POST /api/v1/portfolios

Creates a new portfolio for the authenticated user.

**Request Headers**:
| Header | Required | Description |
|--------|----------|-------------|
| `Authorization` | Yes | JWT bearer token |

**Request Body**:
```json
{
  "name": "My Investment Portfolio",
  "description": "Primary holdings",
  "walletIds": [
    "550e8400-e29b-41d4-a716-446655440000",
    "660e8400-e29b-41d4-a716-446655440001"
  ]
}
```

**Response**: `201 Created`
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "My Investment Portfolio",
  "description": "Primary holdings",
  "walletIds": [
    "550e8400-e29b-41d4-a716-446655440000",
    "660e8400-e29b-41d4-a716-446655440001"
  ]
}
```

**Validation Rules**:
- `name`: Required, max 100 characters
- `description`: Optional, max 500 characters
- `walletIds`: Optional, max 10 wallets per portfolio
- All wallet IDs must exist and belong to user

**Errors**:
| Code | HTTP | Description |
|------|------|-------------|
| `0300003` | 400 | Portfolio limit exceeded (max 10 per user) |
| `0300004` | 400 | Wallet limit exceeded (max 10 per portfolio) |

---

### List Portfolios

#### GET /api/v1/portfolios

Lists all portfolios for the authenticated user.

**Request Headers**:
| Header | Required | Description |
|--------|----------|-------------|
| `Authorization` | Yes | JWT bearer token |

**Response**: `200 OK`
```json
[
  {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "My Investment Portfolio",
    "description": "Primary holdings",
    "isDefault": true,
    "walletCount": 2
  },
  {
    "id": "660e8400-e29b-41d4-a716-446655440001",
    "name": "Trading Portfolio",
    "description": "Short-term positions",
    "isDefault": false,
    "walletCount": 1
  }
]
```

**Response Fields**:
- `isDefault`: Boolean flag for primary portfolio
- `walletCount`: Number of wallets in portfolio

**Performance**:
- Soft-deleted portfolios excluded from results
- Indexed on `(user_id, deleted_at)` for fast filtering

---

### Get Portfolio

#### GET /api/v1/portfolios/{id}

Retrieves a specific portfolio with full details.

**Path Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Portfolio ID |

**Request Headers**:
| Header | Required | Description |
|--------|----------|-------------|
| `Authorization` | Yes | JWT bearer token |

**Response**: `200 OK`
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "My Investment Portfolio",
  "description": "Primary holdings",
  "isDefault": true,
  "walletIds": [
    "550e8400-e29b-41d4-a716-446655440000",
    "660e8400-e29b-41d4-a716-446655440001"
  ]
}
```

**Errors**:
| Code | HTTP | Description |
|------|------|-------------|
| `0300001` | 404 | Portfolio not found |
| `0300002` | 403 | User does not own portfolio |

---

### Update Portfolio

#### PUT /api/v1/portfolios/{id}

Updates portfolio metadata (name, description).

**Path Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Portfolio ID |

**Request Body**:
```json
{
  "name": "Updated Portfolio Name",
  "description": "Updated description"
}
```

**Response**: `200 OK`
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Updated Portfolio Name",
  "description": "Updated description"
}
```

**Notes**:
- Wallet associations are NOT updated via this endpoint (use wallet management endpoints)
- `updatedAt` timestamp automatically set

**Errors**:
| Code | HTTP | Description |
|------|------|-------------|
| `0300001` | 404 | Portfolio not found |
| `0300002` | 403 | User does not own portfolio |

---

### Delete Portfolio

#### DELETE /api/v1/portfolios/{id}

Soft-deletes a portfolio. Historical snapshots are retained.

**Path Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Portfolio ID |

**Response**: `204 No Content`

**Behavior**:
- Sets `deletedAt` timestamp (soft delete)
- Portfolio excluded from future list/get operations
- Snapshots retained for historical reference
- Wallet associations remain in DB (orphaned)

**Errors**:
| Code | HTTP | Description |
|------|------|-------------|
| `0300001` | 404 | Portfolio not found |
| `0300002` | 403 | User does not own portfolio |

---

## Wallet Management Endpoints

### Add Wallets to Portfolio

#### POST /api/v1/portfolios/{id}/wallets

Adds one or more wallets to an existing portfolio.

**Path Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Portfolio ID |

**Request Body**:
```json
{
  "walletIds": [
    "550e8400-e29b-41d4-a716-446655440000",
    "660e8400-e29b-41d4-a716-446655440001"
  ]
}
```

**Response**: `200 OK`
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "walletIds": [
    "550e8400-e29b-41d4-a716-446655440000",
    "660e8400-e29b-41d4-a716-446655440001",
    "770e8400-e29b-41d4-a716-446655440002"
  ]
}
```

**Validation**:
- Duplicate wallet IDs filtered automatically
- Total wallet count must not exceed 10
- All wallets must belong to user
- Wallet already in portfolio is idempotent (no error)

**Errors**:
| Code | HTTP | Description |
|------|------|-------------|
| `0300001` | 404 | Portfolio not found |
| `0300002` | 403 | User does not own portfolio |
| `0300004` | 400 | Adding wallets exceeds limit (max 10) |
| `0300005` | 400 | Wallet already in portfolio |

---

### Remove Wallet from Portfolio

#### DELETE /api/v1/portfolios/{id}/wallets/{walletId}

Removes a wallet from a portfolio.

**Path Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Portfolio ID |
| `walletId` | UUID | Wallet ID to remove |

**Response**: `204 No Content`

**Behavior**:
- Wallet association removed
- Future aggregations exclude this wallet
- Wallet itself remains in wallet service
- Operation is idempotent (removing non-existent wallet is safe)

**Errors**:
| Code | HTTP | Description |
|------|------|-------------|
| `PORT005` | 404 | Portfolio not found |
| `PORT006` | 403 | User does not own portfolio |

---

## Holdings & Summary Endpoints

### Get Portfolio Summary

#### GET /api/v1/portfolios/{id}/summary

Retrieves aggregated holdings summary across all wallets in portfolio with real-time pricing.

**Path Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Portfolio ID |

**Query Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `chainType` | String | No | Filter by chain (e.g., "ethereum", "polygon"). Omit for all chains |

**Response**: `200 OK`
```json
{
  "portfolioId": "550e8400-e29b-41d4-a716-446655440000",
  "totalValueUsd": "52500.00",
  "change24hUsd": "1250.00",
  "change24hPercent": "2.43",
  "lastUpdated": "2024-02-26T14:30:00Z",
  "holdings": [
    {
      "symbol": "ETH",
      "name": "Ethereum",
      "quantity": "10.5",
      "priceUsd": "2500.00",
      "valueUsd": "26250.00",
      "change24hPercent": "2.50",
      "allocation": "50.00",
      "chains": ["ethereum", "polygon"]
    },
    {
      "symbol": "USDC",
      "name": "USD Coin",
      "quantity": "26250.00",
      "priceUsd": "1.00",
      "valueUsd": "26250.00",
      "change24hPercent": "0.00",
      "allocation": "50.00",
      "chains": ["ethereum", "polygon"]
    }
  ],
  "chainBreakdown": {
    "ethereum": "39375.00",
    "polygon": "13125.00"
  }
}
```

**Response Fields**:
- `totalValueUsd`: Sum of all holdings in USD
- `change24hUsd`: Dollar change in last 24 hours
- `change24hPercent`: Percentage change in last 24 hours
- `allocation`: Percentage of portfolio value for each holding
- `chains`: List of blockchain networks this token exists on in portfolio
- `chainBreakdown`: Total value per blockchain network

**Data Sources**:
- Holdings from Redis cache (updated via Kafka balance events)
- Prices from Redis cache (updated via Kafka price events)
- TTL: Holdings cached for 1 hour, prices for 5 minutes

**Errors**:
| Code | HTTP | Description |
|------|------|-------------|
| `0300001` | 404 | Portfolio not found |
| `0300002` | 403 | User does not own portfolio |

---

### Get Portfolio Holdings

#### GET /api/v1/portfolios/{id}/holdings

Alias for portfolio summary endpoint (returns same data).

**Path Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Portfolio ID |

**Query Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `chainType` | String | No | Filter by chain (e.g., "ethereum", "polygon") |

**Response**: `200 OK` - Same format as `/summary`

**Note**: This endpoint provides semantic clarity for clients querying holdings data. Functionally identical to `/summary`.

---

## Portfolio History Endpoints

### Get Portfolio History

#### GET /api/v1/portfolios/{id}/history

Retrieves historical portfolio value snapshots over a specified period.

**Path Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | UUID | Portfolio ID |

**Query Parameters**:
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `period` | String | No | `7d` | Time period: `7d`, `30d`, `90d`, `1y` |

**Response**: `200 OK`
```json
{
  "period": "7d",
  "startValue": "51250.00",
  "endValue": "52500.00",
  "changeUsd": "1250.00",
  "changePercent": "2.43",
  "highValue": "53000.00",
  "lowValue": "50000.00",
  "dataPoints": [
    {
      "timestamp": "2024-02-19T14:30:00Z",
      "valueUsd": "51250.00"
    },
    {
      "timestamp": "2024-02-20T14:30:00Z",
      "valueUsd": "51750.00"
    },
    {
      "timestamp": "2024-02-26T14:30:00Z",
      "valueUsd": "52500.00"
    }
  ]
}
```

**Response Fields**:
- `period`: Requested time period
- `startValue`: Portfolio value at start of period
- `endValue`: Portfolio value at end of period
- `changeUsd`: Absolute change in USD
- `changePercent`: Percentage change
- `highValue`: Highest portfolio value in period
- `lowValue`: Lowest portfolio value in period
- `dataPoints`: Array of (timestamp, value) snapshots

**Period Mapping**:
| Period | Duration | Snapshot Frequency |
|--------|----------|-------------------|
| `7d` | Last 7 days | Hourly snapshots |
| `30d` | Last 30 days | 6-hourly snapshots |
| `90d` | Last 90 days | Daily snapshots |
| `1y` | Last 365 days | Weekly snapshots |

**Data Sources**:
- Snapshots stored in PostgreSQL `portfolio_snapshots` table
- Created after balance updates (configurable interval, default 60 minutes)
- Indexed on `(portfolio_id, snapshot_time)` for range query performance

**Errors**:
| Code | HTTP | Description |
|------|------|-------------|
| `PORT005` | 404 | Portfolio not found |
| `PORT006` | 403 | User does not own portfolio |
| `PORT007` | 400 | Invalid period (must be 7d, 30d, 90d, or 1y) |

---

## Error Response Format

All errors follow this format:

```json
{
  "errorCode": "PORT001",
  "errorMessage": "Invalid portfolio name",
  "timestamp": "2024-02-26T14:30:00Z",
  "path": "/api/v1/portfolios",
  "requestId": "abc-123-def"
}
```

---

## Data Types

### UUID Format

All portfolio and wallet IDs use UUID v4 format: `550e8400-e29b-41d4-a716-446655440000`

### Decimal Precision

All monetary values (prices, quantities, totals) use string representation of BigDecimal for precision:
- `"2500.00"` - price values
- `"10.5"` - token quantities
- `"26250.00"` - portfolio values

### Timestamps

All timestamps use ISO 8601 format with UTC timezone: `2024-02-26T14:30:00Z`

### Blockchain Chain Types

Supported chain values:
- `ethereum` - Ethereum mainnet
- `polygon` - Polygon (Matic)
- `arbitrum` - Arbitrum One
- `optimism` - Optimism
- `base` - Base (Coinbase L2)
- `bsc` - Binance Smart Chain
- `avalanche` - Avalanche C-Chain

---

## Authentication

All endpoints require JWT bearer token in `Authorization` header:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

Token validated by security configuration. User ID extracted from JWT `sub` (subject) claim and verified for portfolio ownership.

---

## Rate Limiting

| Endpoint Group | Limit | Window |
|---|---|---|
| `/api/v1/portfolios` (CRUD) | 30 | 1 minute |
| `/api/v1/portfolios/{id}/summary` | 60 | 1 minute |
| `/api/v1/portfolios/{id}/history` | 60 | 1 minute |
| `/api/v1/portfolios/{id}/wallets` | 20 | 1 minute |

*Note: Rate limiting is enforced at gateway level, not in this service.*

