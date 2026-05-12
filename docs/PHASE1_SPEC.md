# Phase 1 Spec: Foundation (Auth, Wallet, P2P, Multi-Protocol)

## Objective
Xây dựng nền tảng (Foundation) cho VaultX với kiến trúc Đa giao thức (GraphQL, gRPC, REST).
Phase 1 tập trung vào:
1. **API Gateway / BFF:** GraphQL endpoint duy nhất cho frontend.
2. **Identity Service:** Đăng ký, đăng nhập (REST/GraphQL), và xác thực JWT. Expose gRPC server để nội bộ lấy thông tin user.
3. **Wallet Service:** Tạo ví VND mặc định, xem số dư (gRPC/GraphQL).
4. **Transfer Service:** Chuyển tiền nội bộ (P2P cùng loại tiền - VND) sử dụng gRPC để giao tiếp với Wallet.
5. **C++ FX Engine:** Thiết lập TCP Server cơ bản nhận rate và publish lên Kafka.

Mục tiêu là thiết lập xong bộ khung (scaffolding), luồng giao tiếp gRPC/GraphQL cơ bản, và CI/CD flow, tạo tiền đề để các phase sau phát triển các feature phức tạp hơn.

## Tech Stack
- **BFF (Gateway):** Spring Boot 4 + Spring GraphQL (Java 21).
- **Backend Services:** Spring Boot 4, Spring Data JPA, gRPC (Protobuf), Resilience4j.
- **FX Engine:** C++17, Boost.Asio, librdkafka.
- **Database:** PostgreSQL 15+.
- **Messaging:** Apache Kafka (KRaft).
- **Caching:** Redis.
- **Frontend:** React 18, Apollo Client (hoặc urql).

## Commands
```bash
# Build Protobuf (tại thư mục proto/)
mvn clean install -pl proto-module

# Build tất cả Java services
mvn clean install

# Chạy Docker Compose (Kafka, Postgres, Redis, C++ Engine)
docker-compose up -d

# Start BFF Gateway
cd services/api-gateway && mvn spring-boot:run
```

## Project Structure
```text
/
├── proto-module/              → Định nghĩa gRPC Protocol Buffers dùng chung (.proto)
├── services/
│   ├── api-gateway/           → Spring GraphQL BFF (Backend For Frontend)
│   ├── identity-service/      → Auth, Users (REST + gRPC Server)
│   ├── wallet-service/        → Ledger, Balances (gRPC Server)
│   ├── transfer-service/      → P2P Transfers (gRPC Server/Client)
│   └── fx-engine/             → C++17 Rate Engine
├── frontend/                  → React App
└── docker-compose.yml         → Cấu hình hạ tầng cục bộ
```

## Protocol Contracts (Example)

**GraphQL (BFF) - `schema.graphqls`**
```graphql
type Query {
    me: User!
    wallets: [Wallet!]!
}

type Mutation {
    transfer(input: TransferInput!): TransferResult!
}

input TransferInput {
    recipientEmail: String!
    amount: Float!
    currency: String!
    idempotencyKey: String!
}
```

**gRPC (Internal) - `wallet.proto`**
```protobuf
syntax = "proto3";
package vaultx.wallet;

service WalletService {
  rpc GetBalance (BalanceRequest) returns (BalanceResponse);
  rpc ExecuteTransfer (TransferRequest) returns (TransferResponse);
}

message BalanceRequest {
  string user_id = 1;
  string currency_code = 2;
}

message BalanceResponse {
  double available_balance = 1;
}
```

## Code Style
- **Java:** Sử dụng MapStruct cho mapping DTO <-> Entity. Không dùng `@Autowired` field injection, ưu tiên Constructor injection với `@RequiredArgsConstructor` (Lombok).
- **GraphQL:** Tuân thủ chuẩn đặt tên camelCase cho fields/arguments, PascalCase cho Types.
- **gRPC:** Tên service/message PascalCase, tên field snake_case.

## Testing Strategy
- **Unit Tests:** JUnit 5 + Mockito cho các logic tính toán, mapping (đặc biệt MapStruct).
- **Integration Tests:** Testcontainers để spin up Postgres/Kafka.
- **gRPC Tests:** Dùng grpc-testing để mock các gRPC client/server.
- **Coverage Requirement:** Tối thiểu 70% line coverage cho layer Service.

## Boundaries
- **Always do:** Khởi tạo Idempotency Key từ Client cho các thao tác thay đổi trạng thái (mutation). Check JWT token tại GraphQL Gateway và pass thông tin user xuống các backend via gRPC metadata/headers.
- **Ask first:** Thay đổi file `.proto` vì nó ảnh hưởng đến contract của toàn hệ thống. Thay đổi schema GraphQL.
- **Never do:** Gọi trực tiếp Database của service khác. Tất cả truy xuất phải qua gRPC. Update số dư ví trực tiếp (bắt buộc dùng Ledger entry DEBIT/CREDIT).

## Success Criteria
- [ ] Chạy được `docker-compose up` bao gồm PostgreSQL, Redis, Kafka.
- [ ] Dịch dịch các file `.proto` chung thành code Java.
- [ ] Khởi chạy được BFF Gateway (có GraphiQL interface).
- [ ] Identity Service cấp phát được JWT qua REST hoặc GraphQL mutation.
- [ ] Thực hiện thành công 1 GraphQL query `me` gọi qua gRPC tới Identity Service.
- [ ] Thực hiện thành công 1 GraphQL mutation `transfer` gọi qua gRPC tới Transfer Service (cùng loại tiền, có trừ/cộng tiền ở Wallet Service via gRPC).
- [ ] C++ Engine build và connect được tới Kafka.

## Decisions
- **gRPC Timestamp Format:** Dùng `int64` milliseconds (Unix timestamp). Không dùng Google Protobuf `Timestamp` để giữ đơn giản và tránh dependency thêm.
- **gRPC Health Check:** Có. Implement `grpc.health.v1` Health Checking Protocol trong tất cả gRPC service để K8s/Docker check liveness/readiness.