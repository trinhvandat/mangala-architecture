# mangala-wallet-service Design

## Architecture Overview

The service follows a **feature-first layered architecture** with clean separation of concerns.

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
│  External service adapters (Web3j, Kafka)                    │
└─────────────────────────────────────────────────────────────┘
```

---

## Chain Adapter Pattern

### Class Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          <<interface>>                                   │
│                          ChainAdapter                                    │
├─────────────────────────────────────────────────────────────────────────┤
│ + getChainType(): ChainType                                             │
│ + isValidAddress(address: String): boolean                              │
│ + normalizeAddress(address: String): String                             │
│ + getNativeBalance(address: String): Balance                            │
│ + getTokenBalances(address: String, tokens: List<String>): List<Token>  │
│ + getGasPrice(): BigDecimal                                             │
│ + isAvailable(): boolean                                                │
└─────────────────────────────────────────────────────────────────────────┘
                                    △
                                    │ implements
                                    │
┌─────────────────────────────────────────────────────────────────────────┐
│                          EvmChainAdapter                                 │
├─────────────────────────────────────────────────────────────────────────┤
│ - chainType: ChainType                                                  │
│ - web3j: Web3j                                                          │
│ - addressValidator: EvmAddressValidator                                 │
│ - tokenConfig: TokenConfig                                              │
├─────────────────────────────────────────────────────────────────────────┤
│ + getNativeBalance(): uses eth_getBalance                               │
│ + getTokenBalances(): uses eth_call to balanceOf                        │
│ - getErc20Balance(): encodes/decodes ABI                                │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────┐
│       ChainAdapterResolver       │
├─────────────────────────────────┤
│ - adapters: Map<ChainType, ...> │
├─────────────────────────────────┤
│ + getAdapter(chain): Optional   │
│ + getSupportedChains(): List    │
└─────────────────────────────────┘
```

### Extensibility

To add a new chain (e.g., Solana):

1. Create `SolanaChainAdapter implements ChainAdapter`
2. Register as Spring `@Component`
3. Add configuration in `application.yml`
4. ChainAdapterResolver auto-discovers via Spring

---

## Balance Sync Flow

### Sequence Diagram

```
┌───────────┐    ┌───────────────┐    ┌───────────────┐    ┌──────────┐    ┌─────────┐
│ Scheduler │    │ BalanceSync   │    │ ChainAdapter  │    │    DB    │    │  Kafka  │
│           │    │   Service     │    │   (Web3j)     │    │          │    │         │
└─────┬─────┘    └───────┬───────┘    └───────┬───────┘    └────┬─────┘    └────┬────┘
      │                  │                    │                  │               │
      │ @Scheduled(5min) │                    │                  │               │
      │─────────────────>│                    │                  │               │
      │                  │                    │                  │               │
      │                  │ tryLock("balance-sync-lock")          │               │
      │                  │───────────────────────────────────────>               │
      │                  │<───────────────────────────────────────               │
      │                  │ (acquired)                            │               │
      │                  │                    │                  │               │
      │                  │ findAllActiveWallets()                │               │
      │                  │──────────────────────────────────────>│               │
      │                  │<──────────────────────────────────────│               │
      │                  │ List<WalletEntity>                    │               │
      │                  │                    │                  │               │
      │                  │────────────────────────────────────────────────────────
      │                  │       FOR EACH WALLET (batch of 100)  │               │
      │                  │────────────────────────────────────────────────────────
      │                  │                    │                  │               │
      │                  │ getAdapter(chain)  │                  │               │
      │                  │───────────────────>│                  │               │
      │                  │<───────────────────│                  │               │
      │                  │                    │                  │               │
      │                  │ getTokenBalances() │                  │               │
      │                  │───────────────────>│                  │               │
      │                  │                    │ eth_getBalance   │               │
      │                  │                    │ eth_call (ERC20) │               │
      │                  │<───────────────────│                  │               │
      │                  │ List<TokenBalance> │                  │               │
      │                  │                    │                  │               │
      │                  │ upsert balances                       │               │
      │                  │──────────────────────────────────────>│               │
      │                  │<──────────────────────────────────────│               │
      │                  │                    │                  │               │
      │                  │ update wallet.lastSyncedAt            │               │
      │                  │──────────────────────────────────────>│               │
      │                  │                    │                  │               │
      │                  │ publishBalanceUpdate()                │               │
      │                  │─────────────────────────────────────────────────────>│
      │                  │                    │                  │              │
      │                  │────────────────────────────────────────────────────────
      │                  │       END FOR EACH                    │               │
      │                  │────────────────────────────────────────────────────────
      │                  │                    │                  │               │
      │                  │ unlock()           │                  │               │
      │                  │───────────────────────────────────────>               │
      │<─────────────────│                    │                  │               │
      │ SyncResult       │                    │                  │               │
```

### Retry Logic with Exponential Backoff

```
┌──────────────────────────────────────────────────────────────────┐
│                   syncWalletWithRetry()                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Attempt 1: doSyncWallet()                                      │
│       │                                                          │
│       ├── Success ──────────────────────> Return token count     │
│       │                                                          │
│       └── Failure ──> Wait 1000ms                                │
│                           │                                      │
│   Attempt 2: doSyncWallet()                                      │
│       │                                                          │
│       ├── Success ──────────────────────> Return token count     │
│       │                                                          │
│       └── Failure ──> Wait 2000ms                                │
│                           │                                      │
│   Attempt 3: doSyncWallet()                                      │
│       │                                                          │
│       ├── Success ──────────────────────> Return token count     │
│       │                                                          │
│       └── Failure ──> Throw exception (give up)                  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

Configuration:
- MAX_ATTEMPTS: 3
- INITIAL_DELAY_MS: 1000
- Backoff multiplier: 2x (1s → 2s → 4s)
```

---

## ERC-20 Balance Fetching

### Sequence Diagram

```
┌────────────┐    ┌──────────────┐    ┌────────────┐    ┌──────────────┐
│EvmChain    │    │   Web3j      │    │ Blockchain │    │  TokenConfig │
│  Adapter   │    │              │    │    RPC     │    │              │
└─────┬──────┘    └──────┬───────┘    └─────┬──────┘    └──────┬───────┘
      │                  │                  │                  │
      │ getTokenBalances(address)           │                  │
      │─────────────────────────────────────────────────────────
      │                  │                  │                  │
      │ getTokenAddresses(chain)            │                  │
      │────────────────────────────────────────────────────────>│
      │<────────────────────────────────────────────────────────│
      │ List<String> addresses              │                  │
      │                  │                  │                  │
      │──────────────────────────────────────────────────────────
      │   FOR EACH TOKEN ADDRESS            │                  │
      │──────────────────────────────────────────────────────────
      │                  │                  │                  │
      │ encode balanceOf(address)           │                  │
      │ ┌──────────────────────────────┐   │                  │
      │ │ Function: balanceOf          │   │                  │
      │ │ Input: Address (wallet)      │   │                  │
      │ │ Output: Uint256 (balance)    │   │                  │
      │ └──────────────────────────────┘   │                  │
      │                  │                  │                  │
      │ ethCall(token, data)               │                  │
      │─────────────────>│                  │                  │
      │                  │ eth_call         │                  │
      │                  │─────────────────>│                  │
      │                  │<─────────────────│                  │
      │<─────────────────│ 0x...result      │                  │
      │                  │                  │                  │
      │ decode Uint256                      │                  │
      │                  │                  │                  │
      │ getTokenByAddress(chain, token)     │                  │
      │────────────────────────────────────────────────────────>│
      │<────────────────────────────────────────────────────────│
      │ TokenInfo {symbol, name, decimals}  │                  │
      │                  │                  │                  │
      │ convert to human-readable           │                  │
      │ balance = raw / 10^decimals         │                  │
      │                  │                  │                  │
      │──────────────────────────────────────────────────────────
      │   END FOR EACH                      │                  │
      │──────────────────────────────────────────────────────────
      │                  │                  │                  │
      │ Return List<TokenBalance>           │                  │
```

### Token Configuration

Tokens are loaded from `config/tokens.json`:

```json
{
  "ethereum": {
    "chainId": 1,
    "nativeToken": { "symbol": "ETH", "decimals": 18 },
    "tokens": [
      { "symbol": "USDT", "address": "0xdAC17F...", "decimals": 6 },
      { "symbol": "USDC", "address": "0xA0b869...", "decimals": 6 }
    ]
  }
}
```

---

## Distributed Locking

### Redis Lock Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Balance Sync Cluster                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                 │
│  │  Instance 1 │    │  Instance 2 │    │  Instance 3 │                 │
│  │  (Primary)  │    │  (Standby)  │    │  (Standby)  │                 │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘                 │
│         │                  │                  │                         │
│         │    tryLock()     │    tryLock()     │    tryLock()           │
│         │                  │                  │                         │
│         ▼                  ▼                  ▼                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │                         Redis                                │       │
│  │  ┌───────────────────────────────────────────────────────┐  │       │
│  │  │  Key: "balance-sync-lock"                             │  │       │
│  │  │  Value: Instance 1 UUID                               │  │       │
│  │  │  TTL: 10 minutes                                      │  │       │
│  │  └───────────────────────────────────────────────────────┘  │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
│  Result:                                                                │
│  - Instance 1: acquired=true  → runs sync                              │
│  - Instance 2: acquired=false → skips                                  │
│  - Instance 3: acquired=false → skips                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Lock Configuration

```java
RLock lock = redissonClient.getLock("balance-sync-lock");
boolean acquired = lock.tryLock(
    0,           // waitTime: don't wait
    10,          // leaseTime: 10 minutes
    TimeUnit.MINUTES
);
```

**Why 10-minute TTL?**
- Expected sync duration: ~5 minutes for 1000 wallets
- TTL = 2x expected duration (safety margin)
- Auto-releases if instance crashes

---

## Kafka Event Publishing

### Event Schema

```json
{
  "eventId": "uuid-timestamp",
  "eventType": "balance.updates",
  "walletId": "uuid",
  "chain": "ethereum",
  "address": "0x...",
  "balances": [
    {
      "symbol": "ETH",
      "contractAddress": null,
      "amount": "1.234567890123456789",
      "decimals": 18
    },
    {
      "symbol": "USDT",
      "contractAddress": "0xdAC17F...",
      "amount": "1000.000000",
      "decimals": 6
    }
  ],
  "timestamp": "2025-02-25T10:30:00Z"
}
```

### Producer Configuration

```yaml
spring.kafka.producer:
  acks: all              # Wait for all replicas
  retries: 3             # Retry on failure
  enable-idempotence: true  # Exactly-once semantics
```

### Partition Strategy

- **Key**: `walletId` (UUID string)
- **Effect**: All events for same wallet go to same partition
- **Benefit**: Ordering guarantees per wallet

---

## Data Model

### Entity Relationship Diagram

```
┌─────────────────────┐
│       wallets       │
├─────────────────────┤
│ id (PK)             │───────┐
│ user_id             │       │
│ address             │       │
│ chain_type          │       │
│ label               │       │       ┌─────────────────────┐
│ is_active           │       │       │   wallet_balances   │
│ last_synced_at      │       │       ├─────────────────────┤
│ created_at          │       └──────>│ id (PK)             │
│ updated_at          │               │ wallet_id (FK)      │
└─────────────────────┘               │ chain_type          │
                                      │ contract_address    │
                                      │ symbol              │
┌─────────────────────┐               │ name                │
│       tokens        │               │ decimals            │
├─────────────────────┤               │ balance_raw         │
│ id (PK)             │               │ balance             │
│ chain_type          │               │ last_synced_at      │
│ contract_address    │               │ created_at          │
│ symbol              │               │ updated_at          │
│ name                │               └─────────────────────┘
│ decimals            │
│ logo_url            │
│ coingecko_id        │
└─────────────────────┘

Constraints:
- wallets: UNIQUE(user_id, address, chain_type)
- wallet_balances: UNIQUE(wallet_id, chain_type, contract_address)
- tokens: UNIQUE(chain_type, contract_address)
```

### Index Strategy

```sql
-- Wallet lookups
CREATE INDEX idx_wallets_user_id ON wallets(user_id);
CREATE INDEX idx_wallets_address ON wallets(address);

-- Balance queries
CREATE INDEX idx_wallet_balances_wallet_id ON wallet_balances(wallet_id);
CREATE INDEX idx_wallet_balances_contract ON wallet_balances(chain_type, contract_address);
CREATE INDEX idx_wallet_balances_synced_at ON wallet_balances(last_synced_at);

-- Non-zero balance optimization
CREATE INDEX idx_wallet_balances_nonzero ON wallet_balances(wallet_id)
    WHERE balance > 0;
```

---

## Error Handling

### Balance Sync Failure Modes

| Failure | Handling | Impact |
|---------|----------|--------|
| RPC timeout | Retry 3x with backoff | Wallet marked as failed |
| Invalid address | Skip wallet | Logged as warning |
| Token contract error | Skip token, continue | Partial balance |
| Redis unavailable | Skip sync entirely | No sync this cycle |
| Kafka unavailable | Log error, continue | Event not published |

### Error Hierarchy

```
WalletException (base)
├── WalletNotFoundException (WAL-001)
├── WalletAlreadyExistsException (WAL-002)
├── WalletAccessDeniedException (WAL-003)
├── BalanceSyncException
│   ├── RpcTimeoutException
│   └── TokenContractException
└── ChainNotSupportedException
```
