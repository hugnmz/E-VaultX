# Phase 1 Tasks

Dưới đây là danh sách các task chi tiết được chia nhỏ từ `PHASE1_PLAN.md`. Mỗi task được thiết kế để có thể hoàn thành trong 1 session tập trung, với Acceptance Criteria (điều kiện nghiệm thu) và Verification (cách kiểm tra) rõ ràng.

## Step 1: Foundation Setup

- [ ] **Task 1.1: Setup Infrastructure (Docker Compose)**
  - **Acceptance:** File `docker-compose.yml` được tạo ở thư mục root, chứa cấu hình cho PostgreSQL 15, Redis 7, và Apache Kafka (KRaft mode).
  - **Verify:** Chạy `docker-compose up -d` và `docker-compose ps` đảm bảo tất cả các container đều ở trạng thái `Up (healthy)`.
  - **Files:** `docker-compose.yml`

- [ ] **Task 1.2: Khởi tạo Maven Multi-module Project**
  - **Acceptance:** File `pom.xml` ở thư mục root được tạo, định nghĩa các module: `proto-module`, `services/identity-service`, `services/wallet-service`, `services/transfer-service`, `services/api-gateway`. Thiết lập Spring Boot 4 parent.
  - **Verify:** Chạy `mvn clean install` tại thư mục root và xác nhận build SUCCESS.
  - **Files:** `pom.xml`, các file `pom.xml` rỗng (có schema cơ bản) bên trong các thư mục con tương ứng.

- [ ] **Task 1.3: Định nghĩa gRPC Contracts (proto-module)**
  - **Acceptance:** Các file `identity.proto` và `wallet.proto` được định nghĩa với các service/messages chuẩn xác. Cấu hình `protobuf-maven-plugin` trong `pom.xml` của module này.
  - **Verify:** Chạy `mvn clean install -pl proto-module` và kiểm tra thư mục `target/generated-sources/protobuf/grpc-java` có chứa các class Java được gen ra.
  - **Files:** `proto-module/src/main/proto/identity.proto`, `proto-module/src/main/proto/wallet.proto`, `proto-module/pom.xml`

## Step 2: Core Backend Services

- [ ] **Task 2.1: Identity Service - Foundation & Auth REST**
  - **Acceptance:** Cấu hình Spring Boot app (cổng 8081). Setup Flyway migrations cho schema. Khởi tạo JPA entities (`User`, `RefreshToken`). Xây dựng REST endpoints cho `/api/v1/auth/register` và `/api/v1/auth/login` (trả về JWT RS256).
  - **Verify:** Chạy service, dùng curl/Postman để gọi API đăng ký và đăng nhập thành công. Unit tests pass.
  - **Files:** Các file source code Java, `application.yml`, `db/migration/V1__init_identity.sql` trong `services/identity-service`.

- [ ] **Task 2.2: Identity Service - gRPC Server**
  - **Acceptance:** Implement class `IdentityServiceGrpcImpl` kế thừa từ file proto để trả về thông tin user cho nội bộ.
  - **Verify:** Start service và dùng Postman (chế độ gRPC) gửi request gọi hàm lấy thông tin user thành công.
  - **Files:** Source code gRPC bên trong `services/identity-service`.

- [ ] **Task 2.3: Wallet Service - Foundation & Ledger Logic**
  - **Acceptance:** Cấu hình Spring Boot app (cổng 8082). Setup Flyway. Khởi tạo JPA entities. Implement logic kế toán kép (Double-entry ledger) với `@Transactional` và JPA `@Lock(PESSIMISTIC_WRITE)`. Viết API tạo ví VND mặc định (tạm thời qua REST nội bộ hoặc Command).
  - **Verify:** Integration tests kiểm tra invariant của ledger (tổng nợ == tổng có) pass.
  - **Files:** Source code, `application.yml`, `V1__init_wallet.sql` trong `services/wallet-service`.

- [ ] **Task 2.4: Wallet Service - gRPC Server**
  - **Acceptance:** Implement class `WalletServiceGrpcImpl` cung cấp các rpc `GetBalance` và `ExecuteTransfer` dựa trên `wallet.proto`.
  - **Verify:** Dùng Postman test các endpoint gRPC thành công.
  - **Files:** Source code gRPC bên trong `services/wallet-service`.

- [ ] **Task 2.5: C++ FX Engine Base Setup**
  - **Acceptance:** Khởi tạo project CMake. Viết Boost.Asio TCP server skeleton cơ bản. Tích hợp thư viện librdkafka để publish dummy rate lên Kafka.
  - **Verify:** Build bằng CMake, chạy file binary C++. Dùng Kafka CLI (e.g., `kafka-console-consumer.sh`) để xác nhận message đã được đẩy lên topic `fx.rates.raw`.
  - **Files:** `CMakeLists.txt`, `src/main.cpp`, etc. trong `services/fx-engine`.

## Step 3: Complex Backend Service

- [ ] **Task 3.1: Transfer Service - Foundation & Saga Logic (P2P)**
  - **Acceptance:** Khởi tạo Spring Boot app (cổng 8083). Setup Flyway. Viết logic xử lý P2P transfer (cùng tiền tệ VND). Setup gRPC Client để gọi `ExecuteTransfer` sang Wallet Service. Implement Idempotency key logic. **Lưu ý:** Phase 1 chưa dùng Kafka trong Transfer Service — giao tiếp hoàn toàn qua gRPC + `@Transactional`.
  - **Verify:** Dùng Testcontainers chạy Postgres (Kafka được spin up cùng infra stack nhưng Transfer chưa consume/produce — chỉ cần cho FX Engine và infrastructure test), viết test end-to-end cho luồng chuyển tiền P2P thành công và chuyển tiền thất bại (khi không đủ số dư).
  - **Files:** Source code trong `services/transfer-service`.

## Step 4: External API Gateway

- [ ] **Task 4.1: API Gateway (BFF) - GraphQL Setup & Authentication**
  - **Acceptance:** Khởi tạo Spring Boot GraphQL (cổng 8080). Định nghĩa file `schema.graphqls`. Cấu hình filter kiểm tra JWT trên HTTP Header.
  - **Verify:** Start BFF, truy cập trang GraphiQL. Gửi request không có token bị từ chối `401 Unauthorized`.
  - **Files:** `services/api-gateway/src/main/resources/graphql/schema.graphqls`, config security.

- [ ] **Task 4.2: API Gateway (BFF) - Resolvers**
  - **Acceptance:** Cấu hình các gRPC clients để giao tiếp với Identity, Wallet, Transfer. Viết các Query/Mutation resolvers: `me`, `wallets`, `transfer`.
  - **Verify:** Dùng GraphiQL, gắn token hợp lệ, query thông tin `me`, truy vấn số dư ví, và thử gọi mutation `transfer` hoạt động thông suốt từ BFF -> Transfer -> Wallet.
  - **Files:** Các Datafetchers (Resolvers) trong `services/api-gateway`.
