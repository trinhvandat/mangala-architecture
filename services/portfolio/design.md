# mangala-portfolio Design

## Architecture Overview

The service follows a **feature-first layered architecture** with clean separation of concerns. Three independent features (Portfolio Management, Holdings Aggregation, Portfolio History) are coordinated through event-driven updates.

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

## Portfolio Management Flow

### Create Portfolio

```
┌────────┐          ┌─────────────────────┐          ┌────────┐
│ Client │          │ PortfolioController │          │   DB   │
│        │          │                     │          │        │
└───┬────┘          └──────┬──────────────┘          └───┬────┘
    │                      │                            │
    │ POST /portfolios     │                            │
    │ {name, walletIds}    │                            │
    │─────────────────────>│                            │
    │                      │                            │
    │                      │ Check user ownership       │
    │                      │ Validate wallet IDs        │
    │                      │ Check wallet limit         │
    │                      │                            │
    │                      │ Create portfolio           │
    │                      │──────────────────────────>│
    │                      │<──────────────────────────│
    │                      │                            │
    │                      │ Link wallets               │
    │                      │──────────────────────────>│
    │                      │<──────────────────────────│
    │                      │                            │
    │<─────────────────────│                            │
    │ {id, name, wallets}  │                            │
```

**Key Components**:

**CreatePortfolioUseCaseImpl**:
1. Extracts user ID from JWT principal
2. Validates wallet count <= MAX_WALLETS_PER_PORTFOLIO
3. Creates PortfolioEntity with user ID, name, description
4. Adds wallet associations via PortfolioWalletEntity
5. Returns portfolio response

**Portfolio Soft Delete**:
- Portfolios are soft-deleted (deletedAt timestamp set)
- Hard deletion prevents foreign key violations with snapshots
- Queries filter out deleted portfolios

---

## Holdings Aggregation & Real-Time Sync

### Event-Driven Balance Updates

```
┌────────────────┐          ┌──────────────────────┐          ┌──────────────────┐
│ Wallet Service │          │ Kafka Topic          │          │ Portfolio Service│
│                │          │ balance.updates      │          │                  │
└───┬────────────┘          └────────┬─────────────┘          └────────┬─────────┘
    │                                 │                              │
    │ Balance change detected          │                              │
    │ (from blockchain)                │                              │
    │                                  │                              │
    │ Publish BalanceUpdateEvent       │                              │
    │ {walletId, userId, chainType,    │                              │
    │  balances[], timestamp}          │                              │
    │─────────────────────────────────>│                              │
    │                                  │                              │
    │                                  │ BalanceUpdateConsumer        │
    │                                  │ receives event               │
    │                                  │──────────────────────────────>│
    │                                  │                              │
    │                                  │                              │
    │                                  │                    1. Find all portfolios
    │                                  │                       with this wallet
    │                                  │                    2. Aggregate holdings
    │                                  │                       across all wallets
    │                                  │                    3. Cache result in Redis
    │                                  │                    4. Schedule snapshot if
    │                                  │                       interval elapsed
```

### Holdings Aggregation Algorithm

```
GetPortfolioSummary(portfolioId, userId, chainType)
  │
  ├─> Get portfolio wallets from DB
  │
  ├─> For each wallet:
  │    ├─> Get cached balance (Redis key: wallet:{walletId}:{chainType})
  │    └─> Extract token balances, filter by chainType if specified
  │
  ├─> Aggregate tokens by symbol:
  │    ├─> Sum quantities across wallets
  │    ├─> Calculate total value in USD
  │    ├─> Lookup price from cache (Redis key: price:{symbol}:usd)
  │    └─> Calculate allocation %
  │
  ├─> Calculate portfolio metrics:
  │    ├─> Total value USD = sum of all token values
  │    ├─> Change 24h = sum of token changes weighted by value
  │    ├─> Chain breakdown = group values by chain
  │    └─> Last updated = max timestamp of all tokens
  │
  └─> Return PortfolioSummaryResponse
```

**Key Components**:

**HoldingsAggregationService**:
- `processBalanceUpdate(event)`: Updates holdings cache after balance event
- Stores aggregated portfolio holdings in Redis with TTL
- Triggers snapshot creation if configured interval elapsed

**PriceData Cache**:
- Redis key format: `price:{symbol}:usd`
- Updated by PriceUpdateConsumer from Kafka topic `price.updates`
- TTL: 5 minutes (prices refresh frequently)

**Holdings Cache**:
- Redis key format: `holdings:{portfolioId}:{chainType}`
- Updated on balance change
- TTL: 1 hour (stale reads acceptable)

---

## Portfolio History & Snapshots

### Snapshot Creation

```
┌──────────────────────────────────────┐
│ HoldingsAggregationService           │
│ (after balance update processed)     │
└────────────────┬─────────────────────┘
                 │
                 ├─> Check snapshot interval
                 │   (last snapshot time + INTERVAL_MINUTES)
                 │
                 ├─> Yes: Call PortfolioSnapshotService.createSnapshot()
                 │
                 │   ├─> Get current holdings from aggregation
                 │   ├─> Get current portfolio value
                 │   ├─> Create PortfolioSnapshotEntity
                 │   │   {
                 │   │     portfolioId,
                 │   │     totalValueUsd,
                 │   │     holdings (JSON),
                 │   │     chainBreakdown (JSON),
                 │   │     snapshotTime (now)
                 │   │   }
                 │   ├─> Persist to DB
                 │   └─> Return snapshot
                 │
                 └─> No: Skip snapshot
```

### History Query

```
GetPortfolioHistory(portfolioId, userId, period="7d")
  │
  ├─> Parse period (7d, 30d, 90d, 1y)
  │   └─> Calculate since timestamp
  │
  ├─> Query snapshots:
  │    SELECT * FROM portfolio_snapshots
  │    WHERE portfolio_id = ? AND snapshot_time >= ?
  │    ORDER BY snapshot_time ASC
  │
  ├─> Calculate metrics:
  │    ├─> Start value = first snapshot total_value_usd
  │    ├─> End value = last snapshot total_value_usd
  │    ├─> Change USD = end - start
  │    ├─> Change % = (change / start) * 100
  │    ├─> High = max(total_value_usd) from all snapshots
  │    ├─> Low = min(total_value_usd) from all snapshots
  │    └─> Data points = [(timestamp, valueUsd), ...]
  │
  └─> Return GetHistoryResponse
```

**Snapshot Data Structure** (stored as JSONB):

```json
{
  "holdings": [
    {
      "symbol": "ETH",
      "name": "Ethereum",
      "quantity": "10.5",
      "priceUsd": "2500.00",
      "valueUsd": "26250.00",
      "chains": ["ethereum", "polygon"]
    }
  ],
  "chainBreakdown": {
    "ethereum": "15000.00",
    "polygon": "11250.00"
  }
}
```

---

## Data Model

### Entity Relationship Diagram

```
┌──────────────────────┐       ┌─────────────────────┐
│     portfolios       │       │  portfolio_wallets  │
├──────────────────────┤       ├─────────────────────┤
│ id (PK)              │──┐    │ id (PK)             │
│ user_id (FK)         │  │    │ portfolio_id (FK)   │──┐
│ name                 │  │    │ wallet_id           │  │
│ description          │  │    │ created_at          │  │
│ is_default           │  │    └─────────────────────┘  │
│ created_at           │  │                             │
│ updated_at           │  │                             │
│ deleted_at           │  │                             │
└──────────┬───────────┘  │                             │
           │              │                             │
           │              └────────> (one portfolio)    │
           │                                            │
           │                                            │
           └────────────────> (one user)                │
                                                        │
                             ┌──────────────────────────┘
                             │
                             ├─> (many wallets)
                             │
                    ┌────────▼──────────────┐
                    │ portfolio_snapshots   │
                    ├───────────────────────┤
                    │ id (PK)               │
                    │ portfolio_id (FK)     │──┐
                    │ total_value_usd       │  │
                    │ snapshot_time         │  │
                    │ holdings (JSONB)      │  │
                    │ chain_breakdown (JSON)│  │
                    │ created_at            │  │
                    └───────────────────────┘  │
                                               │
                                    (historical record)
```

**PortfolioEntity**:
- Soft delete via `deletedAt` timestamp
- `wallets` relationship: cascading updates, orphan removal
- `isDefault`: flag for user's primary portfolio
- Timestamps: `createdAt`, `updatedAt` auto-managed

**PortfolioWalletEntity**:
- Composite key: `(portfolio_id, wallet_id)`
- No cascading from wallet side (wallet managed by wallet service)
- Tracks association timestamp

**PortfolioSnapshotEntity**:
- Immutable historical records
- `holdings` stored as JSONB for flexible schema
- `chainBreakdown` map for per-chain analysis
- Indexed on `(portfolio_id, snapshot_time)` for range queries

---

## Security Model

### Access Control

```
┌─────────────────┐
│  JWT Principal  │
│  {sub: userId}  │
└────────┬────────┘
         │
         ├─> Extract userId from JWT.subject
         │
         ├─> For Portfolio operations:
         │   └─> Verify portfolio.user_id == userId
         │       (throw PortfolioAccessDeniedException if mismatch)
         │
         ├─> For Holdings/History:
         │   └─> Verify portfolio ownership before aggregation
         │
         └─> Return 403 Forbidden if unauthorized
```

**Controller-Level Checks**:
- All endpoints require `@AuthenticationPrincipal Jwt jwt`
- User ID extracted: `UUID.fromString(jwt.getSubject())`
- Passed to use cases for ownership verification

**Use Case-Level Checks**:
- Each command/query includes userId
- Verified against entity before returning data
- Throws `PortfolioAccessDeniedException` if mismatch

---

## Kafka Integration

### Topics

| Topic | Producer | Consumer | Payload | Purpose |
|-------|----------|----------|---------|---------|
| `balance.updates` | Wallet Service | Portfolio Service | `BalanceUpdateEvent` | Sync token balances |
| `price.updates` | Price Service | Portfolio Service | `PriceUpdateEvent` | Update token prices |

### Event Schemas

**BalanceUpdateEvent**:
```java
{
  walletId: UUID,
  userId: UUID,
  chainType: String,           // "ethereum", "polygon", etc
  address: String,             // wallet address on chain
  balances: [
    {
      contractAddress: String,
      symbol: String,
      name: String,
      decimals: int,
      balance: BigDecimal
    }
  ],
  timestamp: Instant
}
```

**PriceUpdateEvent**:
```java
{
  symbol: String,
  priceUsd: BigDecimal,
  change24hPercent: BigDecimal,
  timestamp: Instant
}
```

### Consumer Configuration

- **Consumer Group**: `portfolio-service`
- **Auto Offset Reset**: `earliest`
- **Max Concurrent Messages**: configurable (default: 3)
- **Error Handling**: Log and continue (non-fatal failures)

---

## Caching Strategy

### Redis Keys

| Key Pattern | Type | TTL | Updated By |
|------------|------|-----|------------|
| `holdings:{portfolioId}:{chainType}` | String (JSON) | 1 hour | BalanceUpdateConsumer |
| `price:{symbol}:usd` | String (number) | 5 min | PriceUpdateConsumer |
| `portfolio:{portfolioId}:summary` | String (JSON) | 30 min | HoldingsAggregationService |

### Cache Invalidation

- **Balance Update**: Invalidates `holdings:*:{chainType}` keys for affected portfolios
- **Price Update**: Invalidates `price:{symbol}:*` keys
- **Portfolio Changes**: Invalidates `portfolio:{portfolioId}:*` keys
- **TTL Expiration**: Automatic cleanup after TTL

