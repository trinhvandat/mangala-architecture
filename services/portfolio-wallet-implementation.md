# Plan: Portfolio & Wallet Services Implementation

## Overview

Implement the core business logic layer of Mangala: **Wallet Service** and **Portfolio Engine**. These services enable users to track their crypto holdings across EVM-compatible blockchains with a non-custodial architecture (we store public addresses only, users manage their own keys).

## Current State

| Component | Status | Notes |
|-----------|--------|-------|
| mangala-authentication | ✅ Complete | JWT, passkeys, RBAC |
| mangala-gateway | ✅ Complete | Routing, rate limiting, auth |
| mangala-common-security | ✅ Complete | Shared security models |
| Infrastructure | ✅ Complete | PostgreSQL, MongoDB, Redis, Kafka |
| mangala-wallet | ❌ Not started | To be implemented |
| mangala-portfolio | ❌ Not started | To be implemented |

## Architecture Decisions

### Non-Custodial Model
- We **never** store private keys
- Users connect wallets (MetaMask, WalletConnect) and sign transactions client-side
- We store only public addresses for portfolio tracking
- Simpler security model, reduced liability

### Database Strategy
- **PostgreSQL**: User-wallet associations, permissions, metadata
- **MongoDB**: Transaction history, portfolio snapshots (document-oriented, flexible schema)
- **Redis**: Real-time price cache, rate limiting
- **StarRocks**: OLAP analytics for P&L calculations (future phase)

### Extensible Chain Support
- Abstract `ChainAdapter` interface for blockchain interactions
- Initial: EVM chains (Ethereum, BSC, Polygon, Arbitrum)
- Extensible: Solana, Bitcoin, etc. via new adapter implementations

---

## Requirements Summary

### Wallet Service
1. Users can add/remove watched wallet addresses
2. Users can label wallets for organization
3. Support multiple EVM chains per address
4. Validate address format per chain
5. Track wallet balances (native + ERC-20 tokens)
6. Sync wallet data on-demand and periodically

### Portfolio Service
1. Aggregate holdings across all user wallets
2. Calculate total portfolio value in USD
3. Track historical portfolio values
4. Support filtering by chain, token, wallet
5. Provide P&L calculations (unrealized gains)
6. Real-time price updates via WebSocket

---

## Acceptance Criteria

### Wallet Service

| ID | Criterion | Testable |
|----|-----------|----------|
| W1 | POST /api/v1/wallets creates wallet with valid EVM address | Integration test |
| W2 | POST /api/v1/wallets returns 400 for invalid address format | Unit test |
| W3 | GET /api/v1/wallets returns all user's wallets with balances | Integration test |
| W4 | DELETE /api/v1/wallets/{id} removes wallet (soft delete) | Integration test |
| W5 | Wallet sync fetches native + top 100 ERC-20 balances | Integration test |
| W6 | Adding duplicate address returns 409 Conflict | Unit test |
| W7 | Users can only access their own wallets (RBAC enforced) | Security test |

### Portfolio Service

| ID | Criterion | Testable |
|----|-----------|----------|
| P1 | GET /api/v1/portfolios/summary returns total value in USD | Integration test |
| P2 | Portfolio value updates within 60s of price change | Performance test |
| P3 | GET /api/v1/portfolios/holdings returns paginated token list | Integration test |
| P4 | Holdings include quantity, price, value, 24h change | Integration test |
| P5 | GET /api/v1/portfolios/history returns daily snapshots | Integration test |
| P6 | P&L calculation shows unrealized gains per token | Integration test |
| P7 | WebSocket /ws/portfolio streams real-time updates | Integration test |

---

## Implementation Steps

### Phase 1: Wallet Service Foundation

#### Step 1.1: Create mangala-wallet repository
```
mangala-wallet/
├── src/main/java/org/mangala/wallet/
│   ├── wallet/
│   │   ├── adapter/web/WalletController.java
│   │   ├── adapter/repository/WalletRepository.java
│   │   ├── usecase/
│   │   │   ├── CreateWalletUseCase.java
│   │   │   ├── GetUserWalletsUseCase.java
│   │   │   ├── DeleteWalletUseCase.java
│   │   │   └── SyncWalletBalancesUseCase.java
│   │   └── domain/
│   │       ├── WalletEntity.java
│   │       └── WalletNotFoundException.java
│   ├── chain/
│   │   ├── ChainAdapter.java (interface)
│   │   ├── ChainType.java (enum)
│   │   ├── evm/EvmChainAdapter.java
│   │   └── evm/EvmAddressValidator.java
│   ├── balance/
│   │   ├── usecase/FetchBalancesUseCase.java
│   │   └── domain/TokenBalance.java
│   └── shared/
│       ├── config/
│       └── exception/ErrorConstant.java
├── src/main/resources/
│   ├── application.yml
│   └── migration/V1__create_wallets_table.sql
└── build.gradle
```

**Files to create**:
- `WalletEntity.java` - JPA entity for wallet
- `WalletController.java` - REST endpoints
- `ChainAdapter.java` - Abstract interface for chain interactions
- `EvmChainAdapter.java` - EVM implementation using Web3j

**Dependencies**:
- Spring Boot 3.4.x
- Spring Data JPA
- Web3j (EVM interaction)
- mangala-common-security
- mangala-exception

#### Step 1.2: Database Schema (PostgreSQL)

```sql
-- V1__create_wallets_table.sql
CREATE TABLE wallets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    address VARCHAR(100) NOT NULL,
    chain_type VARCHAR(20) NOT NULL,
    label VARCHAR(100),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMP NOT NULL DEFAULT now(),
    last_synced_at TIMESTAMP,
    UNIQUE(user_id, address, chain_type)
);

CREATE INDEX idx_wallets_user_id ON wallets(user_id);
CREATE INDEX idx_wallets_address ON wallets(address);
```

#### Step 1.3: Chain Adapter Interface

```java
public interface ChainAdapter {
    ChainType getChainType();
    boolean isValidAddress(String address);
    NativeBalance getNativeBalance(String address);
    List<TokenBalance> getTokenBalances(String address);
    BigDecimal getGasPrice();
}
```

#### Step 1.4: EVM Chain Adapter

Implement using Web3j:
- Connect to RPC endpoints (Infura, Alchemy, public nodes)
- Fetch ETH/BNB/MATIC balance
- Batch fetch ERC-20 balances via multicall
- Handle rate limiting and retries

#### Step 1.5: Wallet API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | /api/v1/wallets | Add wallet address |
| GET | /api/v1/wallets | List user's wallets |
| GET | /api/v1/wallets/{id} | Get wallet details |
| DELETE | /api/v1/wallets/{id} | Remove wallet |
| POST | /api/v1/wallets/{id}:sync | Force balance sync |

---

### Phase 2: Balance & Token Tracking

#### Step 2.1: Token Registry

Store known token metadata:
```sql
CREATE TABLE tokens (
    id UUID PRIMARY KEY,
    chain_type VARCHAR(20) NOT NULL,
    contract_address VARCHAR(100) NOT NULL,
    symbol VARCHAR(20) NOT NULL,
    name VARCHAR(100),
    decimals INT NOT NULL,
    logo_url VARCHAR(500),
    coingecko_id VARCHAR(100),
    is_verified BOOLEAN DEFAULT false,
    UNIQUE(chain_type, contract_address)
);
```

#### Step 2.2: Balance Storage (MongoDB)

```javascript
// Collection: wallet_balances
{
  walletId: "uuid",
  chainType: "ETHEREUM",
  address: "0x...",
  balances: [
    {
      tokenAddress: "native",
      symbol: "ETH",
      balance: "1.5",
      balanceWei: "1500000000000000000"
    },
    {
      tokenAddress: "0xdAC17F958D2ee523a2206206994597C13D831ec7",
      symbol: "USDT",
      balance: "1000.00",
      balanceRaw: "1000000000"
    }
  ],
  lastUpdated: ISODate("2024-01-15T10:30:00Z")
}
```

#### Step 2.3: Background Sync Job

- Scheduled job every 5 minutes
- Fetch balances for all active wallets
- Publish balance updates to Kafka topic `wallet.balance.updated`
- Handle failures with exponential backoff

---

### Phase 3: Portfolio Service

#### Step 3.1: Create mangala-portfolio repository

```
mangala-portfolio/
├── src/main/java/org/mangala/portfolio/
│   ├── portfolio/
│   │   ├── adapter/web/PortfolioController.java
│   │   ├── usecase/
│   │   │   ├── GetPortfolioSummaryUseCase.java
│   │   │   ├── GetHoldingsUseCase.java
│   │   │   └── CalculatePnLUseCase.java
│   │   └── domain/
│   │       ├── PortfolioSummary.java
│   │       └── Holding.java
│   ├── price/
│   │   ├── adapter/PriceFeedClient.java
│   │   ├── usecase/GetTokenPriceUseCase.java
│   │   └── domain/TokenPrice.java
│   ├── snapshot/
│   │   ├── usecase/CreateSnapshotUseCase.java
│   │   └── domain/PortfolioSnapshot.java
│   └── streaming/
│       └── PortfolioWebSocketHandler.java
```

#### Step 3.2: Portfolio API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1/portfolios/summary | Total portfolio value |
| GET | /api/v1/portfolios/holdings | Token holdings list |
| GET | /api/v1/portfolios/history | Historical snapshots |
| GET | /api/v1/portfolios/pnl | P&L breakdown |
| WS | /ws/portfolio | Real-time updates |

#### Step 3.3: Price Feed Integration

- Integrate with CoinGecko API (free tier: 30 calls/min)
- Cache prices in Redis (TTL: 60s)
- Fallback to Binance API for major tokens
- Publish price updates to Kafka topic `price.updated`

#### Step 3.4: Kafka Consumers

```java
@KafkaListener(topics = "wallet.balance.updated")
public void onBalanceUpdated(WalletBalanceEvent event) {
    // Recalculate portfolio for affected user
    // Broadcast via WebSocket
}

@KafkaListener(topics = "price.updated")
public void onPriceUpdated(PriceUpdateEvent event) {
    // Update cached prices
    // Recalculate affected portfolios
    // Broadcast via WebSocket
}
```

#### Step 3.5: Portfolio Snapshots (MongoDB)

```javascript
// Collection: portfolio_snapshots
{
  userId: "uuid",
  timestamp: ISODate("2024-01-15T00:00:00Z"),
  totalValueUsd: 15420.50,
  holdings: [
    {
      symbol: "ETH",
      quantity: "2.5",
      priceUsd: 2500.00,
      valueUsd: 6250.00
    }
  ],
  snapshotType: "DAILY"
}
```

---

### Phase 4: Gateway Integration

#### Step 4.1: Add Routes to Gateway

```yaml
# mangala-gateway application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: wallet-service
          uri: lb://mangala-wallet
          predicates:
            - Path=/api/v1/wallets/**
          filters:
            - name: AuthorizationFilter
            - name: RateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20

        - id: portfolio-service
          uri: lb://mangala-portfolio
          predicates:
            - Path=/api/v1/portfolios/**
          filters:
            - name: AuthorizationFilter
```

#### Step 4.2: Add Authorization Policies

```sql
-- Insert into api_permissions
INSERT INTO api_permissions (http_method, path_pattern, permission_id, service_name, priority)
VALUES
  ('POST', '/api/v1/wallets', (SELECT id FROM permissions WHERE code = 'wallets:create'), 'wallet-service', 10),
  ('GET', '/api/v1/wallets', (SELECT id FROM permissions WHERE code = 'wallets:read'), 'wallet-service', 10),
  ('DELETE', '/api/v1/wallets/{id}', (SELECT id FROM permissions WHERE code = 'wallets:delete'), 'wallet-service', 10),
  ('GET', '/api/v1/portfolios/**', (SELECT id FROM permissions WHERE code = 'portfolios:read'), 'portfolio-service', 10);
```

---

### Phase 5: Testing & Verification

#### Step 5.1: Unit Tests
- Wallet address validation
- Balance calculation
- P&L computation
- Chain adapter mocking

#### Step 5.2: Integration Tests
- API endpoint tests with Testcontainers
- Kafka consumer tests
- MongoDB snapshot tests

#### Step 5.3: E2E Tests
- Full flow: Add wallet → Sync → View portfolio
- WebSocket subscription test
- Multi-chain portfolio aggregation

---

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| RPC rate limiting | Sync failures | Use multiple RPC providers, implement backoff |
| Price API limits | Stale prices | Cache aggressively, use multiple sources |
| Kafka consumer lag | Delayed updates | Monitor lag, scale consumers |
| MongoDB performance | Slow queries | Index on userId, timestamp; consider sharding |
| Chain reorganization | Incorrect balances | Re-sync on reorg events, handle uncle blocks |

---

## Verification Steps

1. **Wallet CRUD**: Create, list, delete wallets via API
2. **Balance Sync**: Add wallet, trigger sync, verify balances fetched
3. **Portfolio Calculation**: Add multiple wallets, verify aggregated value
4. **Real-time Updates**: Subscribe to WebSocket, verify price updates reflected
5. **Authorization**: Verify users cannot access other users' wallets
6. **Performance**: Sync 100 wallets in < 60 seconds

---

## Estimated Effort

| Phase | Description | Effort |
|-------|-------------|--------|
| Phase 1 | Wallet Service Foundation | 3-4 days |
| Phase 2 | Balance & Token Tracking | 2-3 days |
| Phase 3 | Portfolio Service | 3-4 days |
| Phase 4 | Gateway Integration | 1 day |
| Phase 5 | Testing & Verification | 2-3 days |
| **Total** | | **11-15 days** |

---

## Dependencies

- Web3j library for EVM interaction
- CoinGecko/Binance API keys
- Kafka topics: `wallet.balance.updated`, `price.updated`
- MongoDB collections: `wallet_balances`, `portfolio_snapshots`
- Redis for price caching

---

## Future Extensions

1. **Solana Support**: Add `SolanaChainAdapter` implementing `ChainAdapter`
2. **Bitcoin Support**: Add `BitcoinChainAdapter` for BTC tracking
3. **DeFi Positions**: Track LP positions, staking, lending
4. **NFT Support**: Track NFT holdings and floor prices
5. **Tax Reporting**: Calculate cost basis and realized gains
6. **Alerts**: Price alerts, large transaction notifications
