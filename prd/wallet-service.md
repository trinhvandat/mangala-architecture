# PRD: Wallet Service

**Document Version**: 1.0
**Status**: Draft
**Author**: Product/BA Team
**Last Updated**: 2024-02-24

---

## 1. Executive Summary

### 1.1 Product Vision
Enable Mangala users to track their cryptocurrency holdings across multiple blockchain networks through a non-custodial wallet tracking service. Users connect their existing wallet addresses (from MetaMask, hardware wallets, etc.) to monitor balances without exposing private keys.

### 1.2 Business Objectives
| Objective | Success Metric | Target |
|-----------|----------------|--------|
| User wallet adoption | % of users with 1+ wallet added | 80% within 30 days of registration |
| Multi-chain coverage | Chains supported | 4 EVM chains at launch |
| Data freshness | Balance sync latency | < 5 minutes |
| User retention | DAU/MAU ratio | > 40% |

### 1.3 Target Users
- **Primary**: Gen Z crypto investors (18-30) tracking DeFi portfolios
- **Secondary**: Active traders monitoring multiple wallets
- **Tertiary**: Long-term holders checking balances occasionally

---

## 2. Problem Statement

### 2.1 Current Pain Points
1. **Fragmented View**: Users have assets across multiple chains/wallets with no unified view
2. **Manual Tracking**: Users manually check balances on block explorers
3. **No Historical Data**: Difficult to track portfolio growth over time
4. **Security Concerns**: Some portfolio trackers require private key access

### 2.2 Solution
A non-custodial wallet tracking service that:
- Aggregates balances from multiple wallet addresses
- Supports multiple EVM-compatible blockchains
- Syncs balances automatically in near real-time
- Never requires or stores private keys

---

## 3. User Stories

### 3.1 Wallet Management

| ID | As a... | I want to... | So that... | Priority |
|----|---------|--------------|------------|----------|
| US-W01 | User | Add a wallet address by pasting it | I can start tracking that wallet | P0 |
| US-W02 | User | Connect wallet via WalletConnect | I can add without copy-pasting | P1 |
| US-W03 | User | See a list of all my tracked wallets | I know what I'm monitoring | P0 |
| US-W04 | User | Label my wallets (e.g., "Hot Wallet", "Savings") | I can organize them | P1 |
| US-W05 | User | Remove a wallet from tracking | I can declutter my view | P0 |
| US-W06 | User | See which chains a wallet is active on | I know where my assets are | P1 |
| US-W07 | User | Add the same address for multiple chains | I can track cross-chain presence | P0 |

### 3.2 Balance Tracking

| ID | As a... | I want to... | So that... | Priority |
|----|---------|--------------|------------|----------|
| US-B01 | User | See native token balance (ETH, BNB, etc.) | I know my gas reserves | P0 |
| US-B02 | User | See ERC-20 token balances | I know my token holdings | P0 |
| US-B03 | User | See token values in USD | I understand my wealth | P0 |
| US-B04 | User | Manually refresh balances | I get latest data on demand | P1 |
| US-B05 | User | See when balances were last updated | I know data freshness | P1 |
| US-B06 | User | Hide small/dust balances | I focus on meaningful holdings | P2 |

---

## 4. Functional Requirements

### 4.1 Wallet Address Management

#### FR-W01: Add Wallet Address
- **Input**: Wallet address string, chain type, optional label
- **Validation**:
  - Address format valid for selected chain type
  - Address not already added by this user for this chain
  - Maximum 50 wallets per user
- **Output**: Created wallet record with ID
- **Error Codes**:
  - `INVALID_ADDRESS_FORMAT` - Address doesn't match chain format
  - `WALLET_ALREADY_EXISTS` - Duplicate address+chain for user
  - `WALLET_LIMIT_EXCEEDED` - User has 50+ wallets

#### FR-W02: List User Wallets
- **Input**: User ID (from JWT), pagination params
- **Output**: Paginated list of wallets with:
  - Wallet ID, address, chain type, label
  - Last synced timestamp
  - Native balance (if available)
- **Sorting**: By created date (default), by label, by chain

#### FR-W03: Get Wallet Details
- **Input**: Wallet ID
- **Authorization**: User must own the wallet
- **Output**: Wallet details + all token balances

#### FR-W04: Update Wallet
- **Input**: Wallet ID, updated label
- **Authorization**: User must own the wallet
- **Output**: Updated wallet record

#### FR-W05: Delete Wallet
- **Input**: Wallet ID
- **Authorization**: User must own the wallet
- **Behavior**: Soft delete (mark inactive, retain for 30 days)
- **Output**: Success confirmation

### 4.2 Balance Synchronization

#### FR-S01: On-Demand Sync
- **Trigger**: User requests sync via API
- **Rate Limit**: Max 1 sync per wallet per 60 seconds
- **Behavior**:
  1. Fetch native token balance from chain
  2. Fetch ERC-20 balances (top tokens by liquidity)
  3. Store balances in database
  4. Update last_synced_at timestamp
- **Timeout**: 30 seconds max

#### FR-S02: Background Sync
- **Trigger**: Scheduled job every 5 minutes
- **Scope**: All active wallets
- **Batching**: Process 100 wallets per batch
- **Failure Handling**: Retry 3 times with exponential backoff

#### FR-S03: Balance Storage
- **Data Stored**:
  - Token contract address (or "native" for chain token)
  - Token symbol and name
  - Raw balance (integer, smallest unit)
  - Formatted balance (decimal, human-readable)
  - USD value at sync time
- **Retention**: Keep 90 days of historical balances

### 4.3 Chain Support

#### FR-C01: Supported Chains (Launch)
| Chain | Chain ID | Native Token | RPC Provider |
|-------|----------|--------------|--------------|
| Ethereum | 1 | ETH | Infura/Alchemy |
| BNB Smart Chain | 56 | BNB | Public RPC |
| Polygon | 137 | MATIC | Polygon RPC |
| Arbitrum | 42161 | ETH | Arbitrum RPC |

#### FR-C02: Chain Adapter Interface
All chain integrations must implement:
```
interface ChainAdapter {
  getChainType(): ChainType
  isValidAddress(address: string): boolean
  getNativeBalance(address: string): Balance
  getTokenBalances(address: string): TokenBalance[]
}
```

---

## 5. Non-Functional Requirements

### 5.1 Performance
| Metric | Requirement |
|--------|-------------|
| API Response Time | p95 < 500ms for list/get operations |
| Sync Latency | < 30 seconds for single wallet sync |
| Throughput | Support 1000 concurrent users |
| Background Sync | Complete full sync of 10K wallets in < 30 minutes |

### 5.2 Reliability
| Metric | Requirement |
|--------|-------------|
| Availability | 99.5% uptime |
| Data Durability | No balance data loss |
| Sync Success Rate | > 95% of sync attempts succeed |

### 5.3 Security
| Requirement | Implementation |
|-------------|----------------|
| No private keys | Never accept or store private keys |
| User isolation | Users can only access their own wallets |
| Rate limiting | Prevent abuse of sync endpoints |
| Input validation | Validate all address formats |

### 5.4 Scalability
- Horizontal scaling via multiple service instances
- Database sharding by user_id (future)
- Redis caching for frequently accessed balances

---

## 6. API Specification

### 6.1 Endpoints

#### POST /api/v1/wallets
Create a new watched wallet.

**Request**:
```json
{
  "address": "0x742d35Cc6634C0532925a3b844Bc9e7595f8fE34",
  "chainType": "ETHEREUM",
  "label": "My Hot Wallet"
}
```

**Response** (201 Created):
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "address": "0x742d35Cc6634C0532925a3b844Bc9e7595f8fE34",
  "chainType": "ETHEREUM",
  "label": "My Hot Wallet",
  "createdAt": "2024-02-24T10:30:00Z",
  "lastSyncedAt": null
}
```

#### GET /api/v1/wallets
List user's wallets.

**Query Parameters**:
- `page` (default: 0)
- `size` (default: 20, max: 100)
- `chainType` (optional filter)

**Response** (200 OK):
```json
{
  "content": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "address": "0x742d35...",
      "chainType": "ETHEREUM",
      "label": "My Hot Wallet",
      "nativeBalance": {
        "symbol": "ETH",
        "balance": "1.5",
        "valueUsd": 3750.00
      },
      "lastSyncedAt": "2024-02-24T10:35:00Z"
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 1
}
```

#### GET /api/v1/wallets/{id}
Get wallet details with balances.

**Response** (200 OK):
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "address": "0x742d35...",
  "chainType": "ETHEREUM",
  "label": "My Hot Wallet",
  "balances": [
    {
      "tokenAddress": "native",
      "symbol": "ETH",
      "name": "Ethereum",
      "balance": "1.5",
      "valueUsd": 3750.00,
      "price": 2500.00,
      "change24h": 2.5
    },
    {
      "tokenAddress": "0xdAC17F958D2ee523a2206206994597C13D831ec7",
      "symbol": "USDT",
      "name": "Tether USD",
      "balance": "1000.00",
      "valueUsd": 1000.00,
      "price": 1.00,
      "change24h": 0.01
    }
  ],
  "totalValueUsd": 4750.00,
  "lastSyncedAt": "2024-02-24T10:35:00Z"
}
```

#### PATCH /api/v1/wallets/{id}
Update wallet label.

**Request**:
```json
{
  "label": "Updated Label"
}
```

#### DELETE /api/v1/wallets/{id}
Remove wallet from tracking.

**Response** (204 No Content)

#### POST /api/v1/wallets/{id}:sync
Trigger manual balance sync.

**Response** (202 Accepted):
```json
{
  "message": "Sync initiated",
  "estimatedCompletionSeconds": 15
}
```

---

## 7. Data Model

### 7.1 PostgreSQL Tables

```sql
-- Wallet metadata
CREATE TABLE wallets (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    address VARCHAR(100) NOT NULL,
    chain_type VARCHAR(20) NOT NULL,
    label VARCHAR(100),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP,
    last_synced_at TIMESTAMP,
    UNIQUE(user_id, address, chain_type)
);

-- Known tokens registry
CREATE TABLE tokens (
    id UUID PRIMARY KEY,
    chain_type VARCHAR(20) NOT NULL,
    contract_address VARCHAR(100) NOT NULL,
    symbol VARCHAR(20) NOT NULL,
    name VARCHAR(100),
    decimals INT NOT NULL,
    logo_url VARCHAR(500),
    coingecko_id VARCHAR(100),
    UNIQUE(chain_type, contract_address)
);
```

### 7.2 MongoDB Collections

```javascript
// wallet_balances - Current balances
{
  _id: ObjectId,
  walletId: UUID,
  userId: UUID,
  chainType: "ETHEREUM",
  address: "0x...",
  balances: [
    {
      tokenAddress: "native",
      symbol: "ETH",
      name: "Ethereum",
      decimals: 18,
      balanceRaw: "1500000000000000000",
      balance: "1.5",
      priceUsd: 2500.00,
      valueUsd: 3750.00
    }
  ],
  totalValueUsd: 4750.00,
  syncedAt: ISODate
}

// balance_history - Historical snapshots (hourly)
{
  _id: ObjectId,
  walletId: UUID,
  timestamp: ISODate,
  totalValueUsd: 4750.00,
  balances: [...] // Same structure as above
}
```

---

## 8. Error Codes

| Code | HTTP Status | Message |
|------|-------------|---------|
| `0200001` | 400 | Invalid wallet address format |
| `0200002` | 409 | Wallet already exists for this chain |
| `0200003` | 404 | Wallet not found |
| `0200004` | 403 | Cannot access wallet owned by another user |
| `0200005` | 429 | Sync rate limit exceeded (wait 60 seconds) |
| `0200006` | 400 | Wallet limit exceeded (max 50) |
| `0200007` | 400 | Unsupported chain type |
| `0200008` | 503 | Chain RPC unavailable |

---

## 9. Dependencies

### 9.1 External Services
| Service | Purpose | Fallback |
|---------|---------|----------|
| Infura | Ethereum RPC | Alchemy, public nodes |
| CoinGecko | Token prices | Binance API |
| Chain RPCs | Balance queries | Multiple providers |

### 9.2 Internal Services
| Service | Dependency Type |
|---------|-----------------|
| mangala-authentication | JWT validation (via gateway) |
| mangala-gateway | Request routing, rate limiting |
| Kafka | Balance update events |
| Redis | Price caching, rate limiting |

---

## 10. Milestones

| Milestone | Description | Target Date |
|-----------|-------------|-------------|
| M1 | Wallet CRUD API complete | Week 1 |
| M2 | EVM chain adapter (Ethereum) | Week 1 |
| M3 | Balance sync (native + ERC-20) | Week 2 |
| M4 | Multi-chain support (BSC, Polygon, Arbitrum) | Week 2 |
| M5 | Background sync job | Week 3 |
| M6 | Integration with gateway | Week 3 |
| M7 | Testing & bug fixes | Week 3-4 |

---

## 11. Open Questions

| # | Question | Owner | Status |
|---|----------|-------|--------|
| 1 | Should we support ENS name resolution? | Product | Open |
| 2 | How to handle tokens not in our registry? | Engineering | Open |
| 3 | Should we show NFT holdings in wallet view? | Product | Deferred to v2 |
| 4 | What's the maximum token count to display? | UX | Open |

---

## 12. Appendix

### A. Address Validation Regex
```
Ethereum/EVM: ^0x[a-fA-F0-9]{40}$
```

### B. Supported Token Standards
- ERC-20 (fungible tokens)
- Native chain tokens (ETH, BNB, MATIC)

### C. Related Documents
- [System Design: Wallet Service](../services/wallet/design.md)
- [API Specification: Wallet Service](../services/wallet/api.md)
- [PRD: Portfolio Service](./portfolio-service.md)
