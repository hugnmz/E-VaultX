# Phase 1 Implementation Plan

## 1. Major Components & Dependencies

Kiến trúc Phase 1 bao gồm các module sau, với luồng phụ thuộc rõ ràng:

- **Infrastructure:** Cung cấp môi trường chạy (PostgreSQL, Kafka, Redis). *Tất cả các service đều phụ thuộc vào đây.*
- **Proto Module (`proto-module`):** Định nghĩa contract gRPC. *Các Java Backend Services phụ thuộc vào module này.*
- **Identity Service:** Cấp JWT, quản lý User. *Độc lập.*
- **Wallet Service:** Quản lý số dư, Ledger. *Độc lập.*
- **Transfer Service:** Xử lý logic chuyển tiền. *Phụ thuộc vào Wallet Service (để trừ/cộng tiền via gRPC) và Identity (gián tiếp via JWT context).*
- **API Gateway (BFF):** GraphQL Server. *Phụ thuộc vào tất cả backend services (Identity, Wallet, Transfer) qua gRPC.*
- **C++ FX Engine:** Publish tỷ giá. *Độc lập, chỉ phụ thuộc Kafka.*

## 2. Implementation Order (Sequential vs. Parallel)

Kế hoạch triển khai được chia thành 4 bước (Steps). Các task trong cùng một bước có thể làm song song. Không chuyển sang bước tiếp theo nếu bước trước chưa hoàn thành (Verification Checkpoint pass).

### Step 1: Foundation Setup (Sequential)
- Cấu hình `docker-compose.yml` (Postgres, Redis, Kafka KRaft).
- Khởi tạo cấu trúc dự án đa module (Maven multi-module).
- Khởi tạo `proto-module`, định nghĩa `identity.proto` và `wallet.proto`.

### Step 2: Core Backend Services (Parallel)
- **Identity Service:** Setup DB (Flyway), JPA Entity, REST Auth endpoint (Register/Login), và gRPC Server (trả về user info).
- **Wallet Service:** Setup DB (Flyway), JPA Entity, Ledger Logic (Double-entry), và gRPC Server (Query số dư, Execute Transfer mutation).
- **C++ FX Engine:** Setup CMake, Boost.Asio TCP server cơ bản, gửi thử dummy data lên Kafka.

### Step 3: Complex Backend Service (Sequential)
- **Transfer Service:** Setup DB (Flyway). Viết logic Saga (P2P VND). Consume gRPC từ Wallet Service để thực hiện trừ/cộng tiền.

### Step 4: External API Gateway (Sequential)
- **GraphQL BFF:** Setup Spring GraphQL. Định nghĩa `schema.graphqls`. Code các GraphQL Resolvers để map queries/mutations thành các lời gọi gRPC xuống Identity, Wallet, Transfer. Xử lý JWT Validation tại gateway.

## 3. Risks & Mitigation

| Risk | Impact | Mitigation Strategy |
| :--- | :--- | :--- |
| **gRPC Contract Mismatch** | Lỗi runtime khi các service giao tiếp | Quản lý tập trung mọi file `.proto` ở một Maven module duy nhất (`proto-module`). Yêu cầu build module này đầu tiên. |
| **Data Inconsistency (Saga)** | Sai lệch số dư (ví dụ trừ tiền xong nhưng lỗi) | Tuân thủ chặt chẽ Double-Entry Ledger. Wallet Service phải lock row (PESSIMISTIC_WRITE) khi update balance. |
| **Kafka Startup Delay** | Java services crash khi Kafka chưa sẵn sàng | Dùng `depends_on` với condition `service_healthy` trong docker-compose. |

## 4. Verification Checkpoints

*   **Checkpoint 1 (Sau Step 1):** `docker-compose ps` hiện tất cả dịch vụ `Up (healthy)`. `mvn clean install` thành công cho `proto-module` (sinh ra các class Java gRPC).
*   **Checkpoint 2 (Sau Step 2):** Identity và Wallet khởi động thành công trên cổng riêng (ví dụ 8081, 8082). Gọi gRPC UI (ví dụ Postman gRPC) test thử endpoint thành công.
*   **Checkpoint 3 (Sau Step 3):** Viết Integration test (Testcontainers) chuyển tiền từ User A sang User B chạy Pass.
*   **Checkpoint 4 (Sau Step 4):** Mở GraphiQL (UI của GraphQL), gửi mutation Login -> lấy Token -> Pass token vào Header -> Gửi query `me` và `wallets` trả về kết quả đúng.
