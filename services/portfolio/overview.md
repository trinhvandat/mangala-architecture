# mangala-portfolio Service Overview

## Purpose

Portfolio aggregation service for Mangala Wallet platform. Provides unified portfolio management, holdings aggregation across multiple wallets, real-time balance sync via Kafka events, and historical portfolio snapshots for performance tracking.

## Technology Stack

| Component | Technology |
|-----------|------------|
| Language | Java 21 |
| Framework | Spring Boot 3.4.2 |
| Database | PostgreSQL |
| Migrations | Flyway |
| Cache | Redis |
| Event Streaming | Kafka |
| Mapping | MapStruct 1.6.3 |
| Build | Gradle |

## Dependencies

### Internal
- `mangala-common-security` - Shared security models
- `mangala-exception` - Exception framework

### External
- PostgreSQL database (schema: `portfolio`)
- Redis cache cluster
- Kafka broker for balance/price updates

## Feature Catalog

| Feature | Status | Endpoints | Key Classes |
|---------|--------|-----------|-------------|
| Portfolio Management | Done | `POST /api/v1/portfolios`<br>`GET /api/v1/portfolios`<br>`GET /api/v1/portfolios/{id}`<br>`PUT /api/v1/portfolios/{id}`<br>`DELETE /api/v1/portfolios/{id}` | `CreatePortfolioUseCaseImpl`<br>`GetPortfolioUseCaseImpl`<br>`ListPortfoliosUseCaseImpl`<br>`UpdatePortfolioUseCaseImpl`<br>`DeletePortfolioUseCaseImpl` |
| Wallet Management | Done | `POST /api/v1/portfolios/{id}/wallets`<br>`DELETE /api/v1/portfolios/{id}/wallets/{walletId}` | `ManagePortfolioWalletsUseCaseImpl` |
| Holdings Aggregation | Done | `GET /api/v1/portfolios/{id}/summary`<br>`GET /api/v1/portfolios/{id}/holdings` | `GetPortfolioSummaryUseCaseImpl`<br>`HoldingsAggregationServiceImpl` |
| Portfolio History | Done | `GET /api/v1/portfolios/{id}/history` | `GetPortfolioHistoryUseCaseImpl`<br>`PortfolioSnapshotService` |
| Real-time Balance Sync | Done | Kafka topics: `balance.updates`, `price.updates` | `BalanceUpdateConsumer`<br>`PriceUpdateConsumer` |

## Package Structure

```
org.mangala.portfolio/
├── portfolio/                     # Portfolio CRUD feature
│   ├── adapter/
│   │   ├── web/                   # PortfolioController, DTOs
│   │   └── repository/            # JPA repositories
│   ├── usecase/                   # Use case interfaces & impls
│   └── domain/                    # PortfolioEntity, exceptions
│
├── holdings/                      # Holdings aggregation feature
│   ├── adapter/
│   │   └── web/                   # HoldingsController
│   ├── usecase/
│   ├── service/                   # HoldingsAggregationService
│   ├── domain/                    # AggregatedHolding, PortfolioSummary
│   └── event/                     # Kafka consumers
│
├── snapshot/                      # Historical snapshots feature
│   ├── adapter/
│   │   ├── web/                   # PortfolioHistoryController
│   │   └── repository/            # PortfolioSnapshotRepository
│   ├── usecase/
│   ├── service/                   # PortfolioSnapshotService
│   └── domain/                    # PortfolioSnapshotEntity
│
└── shared/                        # Shared configuration
    ├── config/                    # SecurityConfig, KafkaConfig, RedisConfig
    └── exception/                 # Custom exceptions
```

## Database Schema

| Table | Purpose |
|-------|---------|
| `portfolios` | Portfolio records with user ownership |
| `portfolio_wallets` | Portfolio-wallet many-to-many mapping |
| `portfolio_snapshots` | Historical portfolio value snapshots |
| `holdings_cache` | Cached aggregated holdings per portfolio |

## Configuration

### Required Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `DB_HOST` | PostgreSQL host | - |
| `DB_PORT` | PostgreSQL port | 5432 |
| `DB_NAME` | Database name | - |
| `DB_SCHEMA` | Schema name | portfolio |
| `DB_USERNAME` | Database user | - |
| `DB_PASSWORD` | Database password | - |
| `REDIS_HOST` | Redis host | - |
| `REDIS_PORT` | Redis port | 6379 |
| `KAFKA_BROKERS` | Kafka bootstrap servers | - |
| `KAFKA_TOPIC_BALANCE_UPDATES` | Balance updates topic | balance.updates |
| `KAFKA_TOPIC_PRICE_UPDATES` | Price updates topic | price.updates |
| `KAFKA_CONSUMER_GROUP_ID` | Consumer group ID | portfolio-service |
| `SNAPSHOT_INTERVAL_MINUTES` | Interval for snapshot creation | 60 |
| `MAX_WALLETS_PER_PORTFOLIO` | Maximum wallets in single portfolio | 10 |
| `MAX_PORTFOLIOS_PER_USER` | Maximum portfolios per user | 10 |

## Related Documentation

- **Design**: `design.md` - Architecture, event flows, data model
- **API**: `api.md` - Endpoint specifications
