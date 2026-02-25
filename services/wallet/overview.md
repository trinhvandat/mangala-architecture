# mangala-wallet-service Overview

## Purpose

The Wallet Service manages user wallet addresses and token balances across multiple EVM-compatible blockchains. It provides:

- Wallet address management (CRUD operations)
- Multi-chain balance fetching (native + ERC-20 tokens)
- Automatic balance synchronization (scheduled)
- Kafka event publishing for downstream services

## Service Details

| Property | Value |
|----------|-------|
| Port | 8081 |
| Database Schema | `wallet` |
| Status | **Active Development** |
| Sprint | Sprint 2 (Balance Sync Engine) |

## Supported Chains

| Chain | Chain ID | Native Token | Status |
|-------|----------|--------------|--------|
| Ethereum | 1 | ETH | ✅ Active |
| BSC | 56 | BNB | ✅ Active |
| Polygon | 137 | MATIC | ✅ Active |
| Arbitrum | 42161 | ETH | ✅ Active |

## Features

### Implemented

| Feature | Description | Sprint |
|---------|-------------|--------|
| Wallet CRUD | Create, read, update, delete wallet addresses | Pre-Sprint |
| Address Validation | EVM address validation and normalization | Pre-Sprint |
| Native Balance | Fetch ETH/BNB/MATIC balance via Web3j | Sprint 2 |
| ERC-20 Balance | Fetch token balances using eth_call | Sprint 2 |
| Balance Sync Scheduler | Automatic sync every 5 minutes | Sprint 2 |
| Distributed Locking | Redis/Redisson lock for sync coordination | Sprint 2 |
| Kafka Events | Publish balance.updates events | Sprint 2 |
| Balance REST API | GET/POST endpoints for balances | Sprint 2 |

### Planned

| Feature | Description | Sprint |
|---------|-------------|--------|
| Transaction History | Fetch and store wallet transactions | Sprint 4 |
| Manual Balance Refresh | On-demand balance update | Sprint 2 ✅ |
| WebSocket Updates | Real-time balance push | Sprint 5 |

## Package Structure

```
org.mangala.wallet/
├── WalletApplication.java
├── wallet/                    # Wallet management feature
│   ├── adapter/
│   │   ├── web/              # REST controllers
│   │   └── repository/       # JPA repositories
│   ├── domain/               # Entities
│   └── usecase/              # Business logic
├── balance/                   # Balance sync feature (Sprint 2)
│   ├── adapter/
│   │   ├── web/              # Balance REST API
│   │   └── repository/       # Balance repository
│   ├── domain/               # WalletBalanceEntity
│   ├── usecase/              # GetBalances, SyncBalance
│   ├── sync/                 # Scheduler, SyncService
│   └── event/                # Kafka publisher
├── chain/                     # Chain adapter abstraction
│   ├── adapter/
│   │   └── evm/              # EVM implementation
│   ├── config/               # Chain properties, TokenConfig
│   └── domain/               # ChainType, Balance, TokenBalance
├── token/                     # Token metadata
│   ├── adapter/repository/
│   └── domain/
└── shared/                    # Shared utilities
    ├── config/               # Redis, Kafka, JPA configs
    ├── constant/
    └── exception/
```

## Dependencies

| Dependency | Purpose |
|------------|---------|
| spring-boot-starter-web | REST API |
| spring-boot-starter-data-jpa | Database access |
| spring-boot-starter-data-redis | Caching, distributed locks |
| spring-kafka | Event publishing |
| web3j-core | Blockchain interaction |
| redisson | Distributed locking |
| flyway | Database migrations |

## Configuration

Key configuration properties in `application.yml`:

```yaml
# Chain RPC endpoints
application.wallet.chains:
  ethereum:
    rpc-url: ${ETH_RPC_URL}
    chain-id: 1
  bsc:
    rpc-url: ${BSC_RPC_URL}
    chain-id: 56

# Balance sync settings
balance.sync:
  enabled: true
  interval-ms: 300000  # 5 minutes
  batch-size: 100
  lock-ttl-minutes: 10

# Kafka
spring.kafka:
  bootstrap-servers: localhost:9092
```

## Database Tables

| Table | Description |
|-------|-------------|
| wallets | User wallet addresses |
| tokens | Token metadata (symbol, decimals) |
| wallet_balances | Token balances per wallet |

## Kafka Topics

| Topic | Direction | Schema |
|-------|-----------|--------|
| balance.updates | Producer | BalanceUpdateEvent (JSON) |

## Related Documentation

- [Design Document](./design.md) - Architecture and sequence diagrams
- [API Documentation](./api.md) - REST API contracts
- [Kafka Schema Decision](../../../.omc/docs/kafka-schema-decision.md)
