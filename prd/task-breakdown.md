# Task Breakdown: Wallet & Portfolio Services

**Project**: Mangala Wallet & Portfolio Implementation
**Status**: Planning Complete
**Total Estimated Effort**: 4-5 weeks

---

## Sprint Overview

| Sprint | Focus | Duration | Status |
|--------|-------|----------|--------|
| Sprint 1 | Wallet Service Foundation | Week 1 | Not Started |
| Sprint 2 | Balance Sync & Price Feed | Week 2 | Not Started |
| Sprint 3 | Portfolio Service | Week 3 | Not Started |
| Sprint 4 | Integration & Testing | Week 4 | Not Started |

---

## Sprint 1: Wallet Service Foundation

### Epic: WS-001 - Repository Setup
| Task ID | Task | Estimate | Assignee | Status |
|---------|------|----------|----------|--------|
| WS-001-1 | Create mangala-wallet repository with Spring Boot 3.4 | 2h | - | TODO |
| WS-001-2 | Configure build.gradle with dependencies (Web3j, Spring Data JPA) | 1h | - | TODO |
| WS-001-3 | Set up application.yml with database and chain configs | 1h | - | TODO |
| WS-001-4 | Add mangala-common-security as submodule | 0.5h | - | TODO |
| WS-001-5 | Add mangala-exception as submodule | 0.5h | - | TODO |
| WS-001-6 | Create Dockerfile and docker-compose.yml for local dev | 1h | - | TODO |

### Epic: WS-002 - Database Schema
| Task ID | Task | Estimate | Assignee | Status |
|---------|------|----------|----------|--------|
| WS-002-1 | Create V1__create_wallets_table.sql migration | 1h | - | TODO |
| WS-002-2 | Create V2__create_tokens_table.sql migration | 1h | - | TODO |
| WS-002-3 | Create WalletEntity JPA entity | 1h | - | TODO |
| WS-002-4 | Create TokenEntity JPA entity | 1h | - | TODO |
| WS-002-5 | Create WalletRepository interface | 0.5h | - | TODO |
| WS-002-6 | Create TokenRepository interface | 0.5h | - | TODO |

### Epic: WS-003 - Chain Adapter Interface
| Task ID | Task | Estimate | Assignee | Status |
|---------|------|----------|----------|--------|
| WS-003-1 | Design ChainAdapter interface | 1h | - | TODO |
| WS-003-2 | Create ChainType enum (ETHEREUM, BSC, POLYGON, ARBITRUM) | 0.5h | - | TODO |
| WS-003-3 | Create Balance and TokenBalance domain models | 1h | - | TODO |
| WS-003-4 | Create ChainAdapterFactory for adapter resolution | 1h | - | TODO |

### Epic: WS-004 - EVM Chain Adapter
| Task ID | Task | Estimate | Assignee | Status |
|---------|------|----------|----------|--------|
| WS-004-1 | Implement EvmAddressValidator | 1h | - | TODO |
| WS-004-2 | Configure Web3j with RPC endpoints (Infura/Alchemy) | 2h | - | TODO |
| WS-004-3 | Implement getNativeBalance() | 2h | - | TODO |
| WS-004-4 | Implement getTokenBalances() with multicall | 4h | - | TODO |
| WS-004-5 | Add retry logic and error handling | 2h | - | TODO |
| WS-004-6 | Write unit tests for EVM adapter | 2h | - | TODO |

### Epic: WS-005 - Wallet CRUD API
| Task ID | Task | Estimate | Assignee | Status |
|---------|------|----------|----------|--------|
| WS-005-1 | Create CreateWalletUseCase interface and impl | 2h | - | TODO |
| WS-005-2 | Create GetUserWalletsUseCase interface and impl | 1h | - | TODO |
| WS-005-3 | Create GetWalletDetailsUseCase interface and impl | 1h | - | TODO |
| WS-005-4 | Create UpdateWalletUseCase interface and impl | 1h | - | TODO |
| WS-005-5 | Create DeleteWalletUseCase interface and impl | 1h | - | TODO |
| WS-005-6 | Create WalletController with all endpoints | 2h | - | TODO |
| WS-005-7 | Create request/response DTOs | 1h | - | TODO |
| WS-005-8 | Create ErrorConstant enum with error codes | 1h | - | TODO |
| WS-005-9 | Write integration tests for CRUD endpoints | 3h | - | TODO |

**Sprint 1 Total: ~40 hours**

---

## Sprint 2: Balance Sync & Price Feed

### Epic: WS-006 - Balance Storage (MongoDB)
| Task ID | Task | Estimate | Assignee | Status |
|---------|------|----------|----------|--------|
| WS-006-1 | Configure MongoDB connection in wallet service | 1h | - | TODO |
| WS-006-2 | Create WalletBalance document model | 1h | - | TODO |
| WS-006-3 | Create WalletBalanceRepository | 1h | - | TODO |
| WS-006-4 | Create BalanceHistory document for historical data | 1h | - | TODO |

### Epic: WS-007 - Balance Sync Use Cases
| Task ID | Task | Estimate | Assignee | Status |
|---------|------|----------|----------|--------|
| WS-007-1 | Create SyncWalletBalancesUseCase interface and impl | 3h | - | TODO |
| WS-007-2 | Add POST /wallets/{id}:sync endpoint | 1h | - | TODO |
| WS-007-3 | Implement rate limiting (1 sync/wallet/60s) | 2h | - | TODO |
| WS-007-4 | Write integration tests for sync | 2h | - | TODO |

### Epic: WS-008 - Background Sync Job
| Task ID | Task | Estimate | Assignee | Status |
|---------|------|----------|----------|--------|
| WS-008-1 | Create scheduled job for balance sync (every 5 min) | 2h | - | TODO |
| WS-008-2 | Implement batch processing (100 wallets/batch) | 2h | - | TODO |
| WS-008-3 | Add retry logic with exponential backoff | 1h | - | TODO |
| WS-008-4 | Publish WalletBalanceUpdated event to Kafka | 2h | - | TODO |

### Epic: PF-001 - Price Feed Service Setup
| Task ID | Task | Estimate | Assignee | Status |
|---------|------|----------|----------|--------|
| PF-001-1 | Create price feed module in portfolio service | 2h | - | TODO |
| PF-001-2 | Integrate CoinGecko API client | 3h | - | TODO |
| PF-001-3 | Integrate Binance API as fallback | 2h | - | TODO |
| PF-001-4 | Implement Redis price caching (60s TTL) | 2h | - | TODO |
| PF-001-5 | Create scheduled job for price updates | 2h | - | TODO |
| PF-001-6 | Publish PriceUpdated events to Kafka | 2h | - | TODO |

### Epic: WS-009 - Token Registry
| Task ID | Task | Estimate | Assignee | Status |
|---------|------|----------|----------|--------|
| WS-009-1 | Seed initial token list (top 100 by market cap) | 2h | - | TODO |
| WS-009-2 | Create token lookup by contract address | 1h | - | TODO |
| WS-009-3 | Add token logo URLs from CoinGecko | 1h | - | TODO |

**Sprint 2 Total: ~36 hours**

---

## Sprint 3: Portfolio Service

### Epic: PS-001 - Repository Setup
| Task ID | Task | Estimate | Assignee | Status |
|---------|------|----------|----------|--------|
| PS-001-1 | Create mangala-portfolio repository | 2h | - | TODO |
| PS-001-2 | Configure dependencies (Spring Cloud Stream, MongoDB) | 1h | - | TODO |
| PS-001-3 | Set up Kafka consumers | 2h | - | TODO |

### Epic: PS-002 - Portfolio Aggregation
| Task ID | Task | Estimate | Assignee | Status |
|---------|------|----------|----------|--------|
| PS-002-1 | Create GetPortfolioSummaryUseCase | 3h | - | TODO |
| PS-002-2 | Create GetHoldingsUseCase with aggregation | 4h | - | TODO |
| PS-002-3 | Implement chain/wallet filtering | 2h | - | TODO |
| PS-002-4 | Create PortfolioController | 2h | - | TODO |
| PS-002-5 | Write integration tests | 3h | - | TODO |

### Epic: PS-003 - Kafka Consumers
| Task ID | Task | Estimate | Assignee | Status |
|---------|------|----------|----------|--------|
| PS-003-1 | Create WalletBalanceUpdatedConsumer | 2h | - | TODO |
| PS-003-2 | Create PriceUpdatedConsumer | 2h | - | TODO |
| PS-003-3 | Implement portfolio recalculation on events | 3h | - | TODO |
| PS-003-4 | Publish PortfolioUpdated events | 2h | - | TODO |

### Epic: PS-004 - Historical Snapshots
| Task ID | Task | Estimate | Assignee | Status |
|---------|------|----------|----------|--------|
| PS-004-1 | Create PortfolioSnapshot document model | 1h | - | TODO |
| PS-004-2 | Create daily snapshot scheduled job | 2h | - | TODO |
| PS-004-3 | Create GetPortfolioHistoryUseCase | 2h | - | TODO |
| PS-004-4 | Add /portfolios/history endpoint | 1h | - | TODO |

### Epic: PS-005 - P&L Calculation
| Task ID | Task | Estimate | Assignee | Status |
|---------|------|----------|----------|--------|
| PS-005-1 | Create CalculatePnLUseCase | 3h | - | TODO |
| PS-005-2 | Implement FIFO cost basis calculation | 3h | - | TODO |
| PS-005-3 | Add /portfolios/pnl endpoint | 1h | - | TODO |

### Epic: PS-006 - WebSocket Streaming
| Task ID | Task | Estimate | Assignee | Status |
|---------|------|----------|----------|--------|
| PS-006-1 | Create WebSocket handler for portfolio updates | 3h | - | TODO |
| PS-006-2 | Implement subscription management | 2h | - | TODO |
| PS-006-3 | Integrate with PortfolioUpdated Kafka events | 2h | - | TODO |
| PS-006-4 | Add authentication for WebSocket connections | 2h | - | TODO |
| PS-006-5 | Write WebSocket integration tests | 2h | - | TODO |

**Sprint 3 Total: ~52 hours**

---

## Sprint 4: Integration & Testing

### Epic: GW-001 - Gateway Integration
| Task ID | Task | Estimate | Assignee | Status |
|---------|------|----------|----------|--------|
| GW-001-1 | Add wallet service routes to gateway | 1h | - | TODO |
| GW-001-2 | Add portfolio service routes to gateway | 1h | - | TODO |
| GW-001-3 | Configure WebSocket upgrade in gateway | 2h | - | TODO |
| GW-001-4 | Add rate limiting for wallet/portfolio routes | 1h | - | TODO |

### Epic: AUTH-001 - Authorization Policies
| Task ID | Task | Estimate | Assignee | Status |
|---------|------|----------|----------|--------|
| AUTH-001-1 | Create permissions: wallets:create, wallets:read, wallets:delete | 1h | - | TODO |
| AUTH-001-2 | Create permissions: portfolios:read | 0.5h | - | TODO |
| AUTH-001-3 | Add API permission policies for wallet endpoints | 1h | - | TODO |
| AUTH-001-4 | Add API permission policies for portfolio endpoints | 1h | - | TODO |
| AUTH-001-5 | Test RBAC enforcement | 2h | - | TODO |

### Epic: TEST-001 - E2E Testing
| Task ID | Task | Estimate | Assignee | Status |
|---------|------|----------|----------|--------|
| TEST-001-1 | E2E: User adds wallet and sees balance | 3h | - | TODO |
| TEST-001-2 | E2E: Portfolio aggregation across wallets | 3h | - | TODO |
| TEST-001-3 | E2E: Real-time portfolio updates via WebSocket | 3h | - | TODO |
| TEST-001-4 | E2E: Multi-chain portfolio view | 2h | - | TODO |
| TEST-001-5 | Performance test: 1000 concurrent users | 3h | - | TODO |

### Epic: DOC-001 - Documentation
| Task ID | Task | Estimate | Assignee | Status |
|---------|------|----------|----------|--------|
| DOC-001-1 | Update mangala-architecture with wallet service docs | 2h | - | TODO |
| DOC-001-2 | Update mangala-architecture with portfolio service docs | 2h | - | TODO |
| DOC-001-3 | Update system-overview.md with new services | 1h | - | TODO |
| DOC-001-4 | Create API documentation (OpenAPI/Swagger) | 2h | - | TODO |

### Epic: DEPLOY-001 - Deployment
| Task ID | Task | Estimate | Assignee | Status |
|---------|------|----------|----------|--------|
| DEPLOY-001-1 | Create Kubernetes manifests for wallet service | 2h | - | TODO |
| DEPLOY-001-2 | Create Kubernetes manifests for portfolio service | 2h | - | TODO |
| DEPLOY-001-3 | Configure CI/CD pipelines (GitHub Actions) | 3h | - | TODO |
| DEPLOY-001-4 | Set up monitoring (Prometheus/Grafana dashboards) | 3h | - | TODO |

**Sprint 4 Total: ~42 hours**

---

## Summary

| Sprint | Estimated Hours | Key Deliverables |
|--------|-----------------|------------------|
| Sprint 1 | ~40h | Wallet service with CRUD API, EVM chain adapter |
| Sprint 2 | ~36h | Balance sync, price feed, Kafka integration |
| Sprint 3 | ~52h | Portfolio aggregation, history, P&L, WebSocket |
| Sprint 4 | ~42h | Gateway integration, E2E tests, deployment |
| **Total** | **~170h** | Complete Wallet & Portfolio services |

---

## Task Status Legend

| Status | Description |
|--------|-------------|
| TODO | Not started |
| IN PROGRESS | Currently being worked on |
| REVIEW | Awaiting code review |
| DONE | Completed and merged |
| BLOCKED | Waiting on dependency |

---

## Dependencies Graph

```
Sprint 1                    Sprint 2                    Sprint 3                    Sprint 4
─────────                   ─────────                   ─────────                   ─────────
[WS-001]─────┐
Repository    │
              │
[WS-002]──────┼───>[WS-006]                                                        [GW-001]
Schema        │    MongoDB                                                          Gateway
              │         │                                                              │
[WS-003]──────┤         │                                                              │
ChainAdapter  │         │                                                              │
              │         ▼                                                              │
[WS-004]──────┼───>[WS-007]───────>[PS-002]─────────────────────────────────────>[TEST-001]
EVMAdapter    │    Sync            Portfolio                                       E2E Tests
              │         │               │                                              │
[WS-005]──────┘         │               │                                              │
CRUD API                ▼               ▼                                              │
                   [WS-008]───────>[PS-003]                                            │
                   BG Sync         Kafka                                               │
                        │               │                                              │
                   [PF-001]             │                                              │
                   Price Feed──────────►│                                              │
                                        ▼                                              │
                                   [PS-004]                                            │
                                   History                                             │
                                        │                                              │
                                   [PS-005]                                            │
                                   P&L                                                 │
                                        │                                              │
                                   [PS-006]────────────────────────────────────────────┘
                                   WebSocket
```

---

## Risk Register

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| RPC rate limits | High | Medium | Use multiple providers, implement backoff |
| CoinGecko API limits | Medium | Medium | Cache aggressively, use Binance fallback |
| MongoDB performance | Low | High | Proper indexing, consider sharding |
| WebSocket scalability | Medium | High | Use Redis pub/sub for horizontal scaling |

---

## Definition of Done

A task is DONE when:
- [ ] Code is written and compiles
- [ ] Unit tests pass (>80% coverage)
- [ ] Integration tests pass
- [ ] Code review approved
- [ ] Documentation updated
- [ ] Merged to develop branch
