# Project Map (E-VaultX)

## Architecture Overview
E-VaultX is a multi-protocol microservices platform featuring a Double-Entry Ledger.
- **External API:** GraphQL Gateway (BFF)
- **Internal Comm:** gRPC (Protocol Buffers)
- **Async Comm:** Apache Kafka (KRaft mode)
- **Data Stores:** PostgreSQL 15, Redis 7

## Modules & Services

### 1. `proto-module/` (Shared Contracts)
Defines the gRPC Protocol Buffers shared across backend services.
- **Key Files:** `identity.proto`, `wallet.proto`
- **Build Command:** `mvn clean install -pl proto-module`

### 2. `services/api-gateway/` (BFF)
Spring GraphQL server serving as the single entry point for the frontend.
- **Responsibilities:** GraphQL Resolvers, JWT Validation, mapping GraphQL requests to internal gRPC calls.
- **Key Files:** `schema.graphqls`

### 3. `services/identity-service/`
Manages user authentication and identity.
- **Responsibilities:** Register/Login (REST/GraphQL), issues JWTs, provides user info via gRPC to other services.
- **Tech:** Spring Boot, Spring Security, REST + gRPC Server, PostgreSQL.

### 4. `services/wallet-service/`
Core financial ledger and balance management.
- **Responsibilities:** Enforces strict Double-Entry accounting (DEBIT/CREDIT). No direct balance updates. Distributed locks (Redisson) & pessimistic DB locks for mutations.
- **Tech:** Spring Boot, gRPC Server, PostgreSQL (3NF, UUID v4).

### 5. `services/transfer-service/`
Handles complex business logic for moving funds (e.g., P2P transfers).
- **Responsibilities:** Orchestrates transfers using Saga pattern, calls Wallet Service via gRPC to commit balance changes, ensures idempotency.
- **Tech:** Spring Boot, gRPC Server/Client.

### 6. `services/fx-engine/`
Foreign exchange rate engine.
- **Responsibilities:** Publishes FX rates to Kafka (`fx:rate:{from}:{to}`).
- **Tech:** C++17, Boost.Asio, librdkafka.

### 7. `frontend/`
User interface.
- **Tech:** React 18, Apollo Client (or urql).

## Infrastructure (`docker-compose.yml`)
- **PostgreSQL 15:** Primary database (managed via Flyway).
- **Redis 7:** Distributed caching & Redisson locks.
- **Kafka 3.x:** Event streaming in KRaft mode.

## Key Technical Rules
- **Idempotency:** Required for all transfer mutations.
- **Mapping:** Use MapStruct for DTO ↔ Entity mapping. No reflection-based tools.
- **Testing:** JUnit 5, Mockito, Testcontainers for integration tests.

## Reference Documents
- **System Requirements:** `VaultX_SRS_v1.1.md`, `docs/VaultX_SRS_v1.2.md`
- **Phase 1 Plan:** `docs/PHASE1_PLAN.md`
- **Phase 1 Spec:** `docs/PHASE1_SPEC.md`
