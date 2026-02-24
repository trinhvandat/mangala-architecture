# PRD: Portfolio Service

**Document Version**: 1.0
**Status**: Draft
**Author**: Product/BA Team
**Last Updated**: 2024-02-24

---

## 1. Executive Summary

### 1.1 Product Vision
Provide Mangala users with a unified, real-time view of their entire cryptocurrency portfolio across all tracked wallets and chains. Enable users to understand their total wealth, track performance over time, and make informed investment decisions.

### 1.2 Business Objectives
| Objective | Success Metric | Target |
|-----------|----------------|--------|
| Portfolio engagement | Daily portfolio views per user | > 3 views/day |
| Real-time adoption | Users with WebSocket connected | > 60% of active users |
| Data accuracy | Price deviation from market | < 1% |
| Performance insights | Users viewing P&L | > 50% weekly |

### 1.3 Target Users
- **Primary**: Active crypto investors tracking multi-chain portfolios
- **Secondary**: Traders monitoring real-time value changes
- **Tertiary**: Tax-conscious users tracking cost basis

---

## 2. Problem Statement

### 2.1 Current Pain Points
1. **No Unified View**: Users must manually aggregate across wallets/chains
2. **Delayed Data**: Many trackers update prices infrequently
3. **No Performance Tracking**: Hard to know if portfolio is growing
4. **Complex P&L**: Difficult to understand unrealized gains

### 2.2 Solution
A portfolio aggregation service that:
- Combines all wallet holdings into one view
- Updates prices in real-time via WebSocket
- Tracks historical portfolio value
- Calculates P&L with clear breakdowns

---

## 3. User Stories

### 3.1 Portfolio Overview

| ID | As a... | I want to... | So that... | Priority |
|----|---------|--------------|------------|----------|
| US-P01 | User | See my total portfolio value in USD | I know my total wealth | P0 |
| US-P02 | User | See portfolio value update in real-time | I react to market changes | P0 |
| US-P03 | User | See 24h change ($ and %) | I know daily performance | P0 |
| US-P04 | User | Filter portfolio by chain | I see chain-specific holdings | P1 |
| US-P05 | User | Filter portfolio by wallet | I see wallet-specific holdings | P1 |

### 3.2 Holdings View

| ID | As a... | I want to... | So that... | Priority |
|----|---------|--------------|------------|----------|
| US-H01 | User | See list of all tokens I hold | I know my asset allocation | P0 |
| US-H02 | User | See quantity, price, and value per token | I understand each holding | P0 |
| US-H03 | User | See 24h price change per token | I know which assets moved | P1 |
| US-H04 | User | Sort holdings by value, change, name | I find specific tokens | P1 |
| US-H05 | User | Search for a specific token | I quickly locate holdings | P2 |

### 3.3 Performance & History

| ID | As a... | I want to... | So that... | Priority |
|----|---------|--------------|------------|----------|
| US-PH01 | User | See portfolio value chart (7d, 30d, 90d, 1y) | I track growth over time | P1 |
| US-PH02 | User | See all-time high/low portfolio value | I know my best/worst | P2 |
| US-PH03 | User | See P&L for each token | I know profitable holdings | P1 |
| US-PH04 | User | See total unrealized P&L | I know overall gain/loss | P1 |

### 3.4 Real-Time Updates

| ID | As a... | I want to... | So that... | Priority |
|----|---------|--------------|------------|----------|
| US-RT01 | User | Receive live price updates | My portfolio stays current | P0 |
| US-RT02 | User | See visual indicators for price changes | I notice movements | P1 |
| US-RT03 | User | Opt out of real-time updates | I save battery/bandwidth | P2 |

---

## 4. Functional Requirements

### 4.1 Portfolio Aggregation

#### FR-PA01: Calculate Total Portfolio Value
- **Input**: User ID
- **Process**:
  1. Fetch all active wallets for user
  2. Get latest balances for each wallet
  3. Get current USD price for each token
  4. Sum: Σ (token_balance × token_price_usd)
- **Output**: Total value in USD with breakdown

#### FR-PA02: Holdings Aggregation
- **Logic**: Combine same tokens across wallets/chains
- **Example**:
  - Wallet A (Ethereum): 1.0 ETH
  - Wallet B (Arbitrum): 0.5 ETH
  - Holdings: ETH = 1.5 (combined)

#### FR-PA03: Chain/Wallet Filtering
- **Filters**:
  - By chain type (ETHEREUM, BSC, etc.)
  - By wallet ID
  - By token (search)
- **Behavior**: Recalculate totals based on filter

### 4.2 Price Feed

#### FR-PF01: Price Data Sources
| Priority | Source | Tokens Covered | Update Frequency |
|----------|--------|----------------|------------------|
| 1 | CoinGecko | Top 5000 | Every 60 seconds |
| 2 | Binance | Trading pairs | Real-time WebSocket |
| 3 | DEX prices | Long-tail tokens | On-demand |

#### FR-PF02: Price Caching
- **Storage**: Redis
- **TTL**: 60 seconds for major tokens, 5 minutes for others
- **Fallback**: Return cached price if API fails

#### FR-PF03: Price Update Events
- **Kafka Topic**: `price.updated`
- **Event Payload**:
```json
{
  "tokenId": "ethereum",
  "symbol": "ETH",
  "priceUsd": 2500.00,
  "change24h": 2.5,
  "timestamp": "2024-02-24T10:30:00Z"
}
```

### 4.3 Historical Data

#### FR-HD01: Portfolio Snapshots
- **Frequency**: Daily at 00:00 UTC
- **Data Stored**:
  - Total portfolio value
  - Individual holding values
  - Token prices at snapshot time
- **Retention**: 2 years

#### FR-HD02: Historical Value Query
- **Periods**: 24h, 7d, 30d, 90d, 1y, all-time
- **Resolution**:
  - 24h: Hourly data points
  - 7d-30d: Daily data points
  - 90d+: Weekly data points

### 4.4 P&L Calculation

#### FR-PL01: Unrealized P&L
- **Formula**: `(current_value - cost_basis) / cost_basis × 100`
- **Cost Basis Methods** (v1: FIFO only):
  - FIFO (First In, First Out)
  - Future: LIFO, Average Cost

#### FR-PL02: Cost Basis Tracking
- **Source**: Transaction history (future integration)
- **MVP**: Use first known balance as cost basis
- **Display**: Per-token and total P&L

### 4.5 Real-Time Streaming

#### FR-RT01: WebSocket Connection
- **Endpoint**: `wss://api.mangala.io/ws/portfolio`
- **Authentication**: JWT token in connection header
- **Heartbeat**: Ping every 30 seconds

#### FR-RT02: Subscription Model
```json
// Subscribe to portfolio updates
{
  "action": "subscribe",
  "channel": "portfolio",
  "filters": {
    "chainType": "ETHEREUM"  // optional
  }
}
```

#### FR-RT03: Update Messages
```json
{
  "type": "portfolio.update",
  "data": {
    "totalValueUsd": 15420.50,
    "change24h": 320.00,
    "change24hPercent": 2.12,
    "updatedAt": "2024-02-24T10:30:15Z"
  }
}

{
  "type": "holding.update",
  "data": {
    "symbol": "ETH",
    "priceUsd": 2505.00,
    "change24h": 2.3,
    "valueUsd": 6262.50,
    "quantity": "2.5"
  }
}
```

---

## 5. Non-Functional Requirements

### 5.1 Performance
| Metric | Requirement |
|--------|-------------|
| Portfolio Summary API | p95 < 200ms |
| Holdings List API | p95 < 300ms |
| Historical Data API | p95 < 500ms |
| WebSocket Latency | < 100ms from price update to client |
| Concurrent WebSocket Connections | Support 10,000 |

### 5.2 Reliability
| Metric | Requirement |
|--------|-------------|
| Availability | 99.9% uptime |
| Price Data Freshness | < 2 minutes stale |
| Snapshot Success Rate | 100% daily snapshots |

### 5.3 Scalability
- Stateless service instances (horizontal scaling)
- Redis cluster for price caching
- Kafka partitioning by user ID for ordered processing

---

## 6. API Specification

### 6.1 REST Endpoints

#### GET /api/v1/portfolios/summary
Get portfolio summary.

**Query Parameters**:
- `chainType` (optional): Filter by chain
- `walletId` (optional): Filter by wallet

**Response**:
```json
{
  "totalValueUsd": 15420.50,
  "change24hUsd": 320.00,
  "change24hPercent": 2.12,
  "chainBreakdown": [
    {"chainType": "ETHEREUM", "valueUsd": 10000.00, "percent": 64.8},
    {"chainType": "BSC", "valueUsd": 5420.50, "percent": 35.2}
  ],
  "lastUpdated": "2024-02-24T10:30:00Z"
}
```

#### GET /api/v1/portfolios/holdings
Get token holdings.

**Query Parameters**:
- `page`, `size`: Pagination
- `chainType`: Filter by chain
- `walletId`: Filter by wallet
- `sort`: `value`, `change24h`, `symbol` (default: `value`)
- `order`: `asc`, `desc` (default: `desc`)
- `search`: Token symbol search

**Response**:
```json
{
  "content": [
    {
      "symbol": "ETH",
      "name": "Ethereum",
      "logoUrl": "https://...",
      "quantity": "2.5",
      "priceUsd": 2500.00,
      "valueUsd": 6250.00,
      "change24hPercent": 2.5,
      "allocation": 40.5,
      "chains": ["ETHEREUM", "ARBITRUM"],
      "wallets": [
        {"walletId": "...", "label": "Hot Wallet", "quantity": "1.5"},
        {"walletId": "...", "label": "Cold Storage", "quantity": "1.0"}
      ]
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 15
}
```

#### GET /api/v1/portfolios/history
Get historical portfolio values.

**Query Parameters**:
- `period`: `24h`, `7d`, `30d`, `90d`, `1y`, `all`

**Response**:
```json
{
  "period": "7d",
  "dataPoints": [
    {"timestamp": "2024-02-17T00:00:00Z", "valueUsd": 14500.00},
    {"timestamp": "2024-02-18T00:00:00Z", "valueUsd": 14800.00},
    // ...
  ],
  "startValue": 14500.00,
  "endValue": 15420.50,
  "changeUsd": 920.50,
  "changePercent": 6.35,
  "highValue": 15800.00,
  "lowValue": 14200.00
}
```

#### GET /api/v1/portfolios/pnl
Get P&L breakdown.

**Response**:
```json
{
  "totalUnrealizedPnlUsd": 2500.00,
  "totalUnrealizedPnlPercent": 19.3,
  "holdings": [
    {
      "symbol": "ETH",
      "quantity": "2.5",
      "costBasis": 2000.00,
      "currentPrice": 2500.00,
      "unrealizedPnlUsd": 1250.00,
      "unrealizedPnlPercent": 25.0
    }
  ]
}
```

### 6.2 WebSocket Endpoint

#### WSS /ws/portfolio

**Connection**:
```
wss://api.mangala.io/ws/portfolio
Headers: Authorization: Bearer <jwt>
```

**Messages**:
```json
// Client -> Server: Subscribe
{"action": "subscribe", "channel": "portfolio"}

// Server -> Client: Subscription confirmed
{"type": "subscribed", "channel": "portfolio"}

// Server -> Client: Portfolio update
{"type": "portfolio.update", "data": {...}}

// Server -> Client: Individual holding update
{"type": "holding.update", "data": {...}}

// Client -> Server: Unsubscribe
{"action": "unsubscribe", "channel": "portfolio"}
```

---

## 7. Data Model

### 7.1 MongoDB Collections

```javascript
// portfolio_snapshots - Daily snapshots
{
  _id: ObjectId,
  userId: UUID,
  date: ISODate("2024-02-24T00:00:00Z"),
  totalValueUsd: 15420.50,
  holdings: [
    {
      symbol: "ETH",
      quantity: "2.5",
      priceUsd: 2500.00,
      valueUsd: 6250.00
    }
  ],
  chainBreakdown: {
    "ETHEREUM": 10000.00,
    "BSC": 5420.50
  }
}

// price_history - Token price history
{
  _id: ObjectId,
  tokenId: "ethereum",
  symbol: "ETH",
  prices: [
    {timestamp: ISODate, priceUsd: 2500.00},
    // Rolling 30 days of hourly prices
  ]
}
```

### 7.2 Redis Keys

```
# Current prices
price:{tokenId} -> {"priceUsd": 2500.00, "change24h": 2.5, "updatedAt": "..."}

# Price cache TTL: 60 seconds

# User portfolio cache (invalidated on balance update)
portfolio:{userId}:summary -> {...}
portfolio:{userId}:holdings -> [...]

# Cache TTL: 30 seconds
```

---

## 8. Kafka Topics

| Topic | Producer | Consumer | Purpose |
|-------|----------|----------|---------|
| `price.updated` | Price Feed Service | Portfolio Service | Trigger portfolio recalculation |
| `wallet.balance.updated` | Wallet Service | Portfolio Service | Trigger portfolio recalculation |
| `portfolio.updated` | Portfolio Service | WebSocket Handler | Push to connected clients |

---

## 9. Error Codes

| Code | HTTP Status | Message |
|------|-------------|---------|
| `0300001` | 404 | Portfolio not found (no wallets) |
| `0300002` | 503 | Price feed unavailable |
| `0300003` | 400 | Invalid period parameter |
| `0300004` | 400 | Invalid filter combination |

---

## 10. Dependencies

### 10.1 Internal Services
| Service | Dependency |
|---------|------------|
| Wallet Service | Balance data |
| Authentication | User identity |
| Gateway | Routing, WebSocket upgrade |

### 10.2 External Services
| Service | Purpose |
|---------|---------|
| CoinGecko | Token prices |
| Binance | Real-time prices |

---

## 11. Milestones

| Milestone | Description | Target |
|-----------|-------------|--------|
| M1 | Portfolio summary API | Week 1 |
| M2 | Holdings aggregation | Week 1 |
| M3 | Price feed integration | Week 1-2 |
| M4 | Historical snapshots | Week 2 |
| M5 | P&L calculation | Week 2-3 |
| M6 | WebSocket streaming | Week 3 |
| M7 | Gateway integration | Week 3 |
| M8 | Testing & optimization | Week 4 |

---

## 12. Open Questions

| # | Question | Status |
|---|----------|--------|
| 1 | Should we show unrealized P&L before cost basis integration? | Open |
| 2 | What's the WebSocket reconnection strategy? | Open |
| 3 | Should we support fiat currencies other than USD? | Deferred |

---

## 13. Related Documents
- [PRD: Wallet Service](./wallet-service.md)
- [System Design: Portfolio Service](../services/portfolio/design.md)
- [Architecture: System Overview](../topology/system-overview.md)
