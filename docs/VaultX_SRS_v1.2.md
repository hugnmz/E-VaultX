**VaultX**

Personal Finance & Multi-Currency Wallet Platform

| Software Requirements Specification (SRS) Version 1.2  ·  IEEE 830 / ISO/IEC 29148 Compliant  ·  Java-First Revision |
| :---: |

| Field | Value |
| ----- | ----- |
| Document Title | VaultX — Software Requirements Specification |
| Version | 1.2 (Roadmap Revised – C++ Early Integration) |
| Status | Draft v1.2 |
| Author | VaultX Engineering Team |
| Technology Stack | Spring Boot 4 · Spring Data JPA · Spring Security · Apache Kafka · C++17 · React 18 |
| Classification | Internal / Confidential |
| Date Created | May 2026 (Revised May 2026) |

# **Table of Contents**

# **1. Introduction**

## **1.1 Purpose**

Tài liệu này là Software Requirements Specification (SRS) cho hệ thống VaultX — một nền tảng ví điện tử đa tiền tệ phục vụ cá nhân. Mục tiêu của tài liệu là mô tả đầy đủ và chính xác các yêu cầu chức năng, phi chức năng, thiết kế cơ sở dữ liệu, kiến trúc hệ thống, và các ràng buộc kỹ thuật để làm nền tảng cho việc phát triển, kiểm thử, và vận hành hệ thống.

Tài liệu được soạn theo tiêu chuẩn IEEE 830-1998 và ISO/IEC 29148:2018. Ngôn ngữ được dùng trong mô tả yêu cầu tránh các phát biểu tuyệt đối như "luôn luôn", "không bao giờ", "hoàn toàn" vì thực tế triển khai phụ thuộc vào nhiều yếu tố bên ngoài.

## **1.2 Scope**

VaultX là nền tảng ví điện tử đa tiền tệ dành cho người dùng cá nhân, bao gồm các nhóm chức năng chính:

* Quản lý ví đa tiền tệ (multi-currency wallet) với cơ chế kế toán kép (double-entry ledger).

* Chuyển tiền nội bộ (P2P) và chuyển đổi ngoại tệ (FX conversion) giữa các ví.

* Cung cấp tỷ giá thời gian thực thông qua FX Rate Engine được xây dựng bằng C++17.

* Phân tích chi tiêu (spending analytics) theo danh mục, thời gian, và ngân sách.

* Phát hiện giao dịch bất thường (fraud detection) theo luật dựa trên quy tắc (rule-based).

* Quản trị hệ thống (admin panel) bao gồm KYC, cấu hình tỷ giá, và giám sát ledger.

VaultX không bao gồm các chức năng: giao dịch cổ phiếu, blockchain/cryptocurrency, tích hợp ngân hàng thực, hoặc xử lý thanh toán bằng thẻ tín dụng thật.

## **1.3 Definitions, Acronyms, and Abbreviations**

| Term / Acronym | Definition |
| ----- | ----- |
| SRS | Software Requirements Specification |
| FX / FX Rate | Foreign Exchange — tỷ giá chuyển đổi giữa hai loại tiền tệ |
| FX Rate Engine | Module C++17 nhận tỷ giá feed, tính toán spread, và publish lên Kafka |
| Double-Entry Ledger | Phương pháp kế toán kép: mỗi giao dịch tạo đúng 2 bản ghi (debit \+ credit) để tổng ledger cân bằng về 0 |
| Saga Pattern | Mẫu thiết kế cho distributed transaction không dùng 2PC; sử dụng compensating transaction khi có lỗi |
| Idempotency Key | Khóa định danh duy nhất cho một request, đảm bảo cùng request gửi nhiều lần chỉ được xử lý một lần |
| Outbox Pattern | Mẫu thiết kế ghi event vào cùng transaction DB, sau đó relay sang Kafka để đảm bảo at-least-once delivery |
| CQRS | Command Query Responsibility Segregation — tách riêng write path và read path |
| KYC | Know Your Customer — quy trình xác minh danh tính người dùng |
| P2P Transfer | Peer-to-peer transfer — chuyển tiền trực tiếp giữa hai người dùng |
| JWT | JSON Web Token — chuẩn xác thực không trạng thái |
| RBAC | Role-Based Access Control — phân quyền theo vai trò |
| HPA | Horizontal Pod Autoscaler — tự động mở rộng số lượng pod trong Kubernetes |
| 3NF | Third Normal Form — dạng chuẩn thứ ba trong thiết kế cơ sở dữ liệu quan hệ |
| Spread | Chênh lệch giữa tỷ giá mua (bid) và tỷ giá bán (ask) của một cặp tiền tệ |
| Mid Rate | Tỷ giá trung tâm \= (bid \+ ask) / 2 |
| Flyway | Database Migration Tool — quản lý versioned SQL migration script; schema thay đổi được track và apply theo thứ tự |
| TCP | Transmission Control Protocol — giao thức truyền tải hướng kết nối có xác nhận (acknowledgment-based) |
| MapStruct | Compile-time DTO↔Entity mapper; không dùng reflection, sinh code Java thuần tại build time; tích hợp với Spring và JPA entity |

## **1.4 References**

* IEEE Std 830-1998 — IEEE Recommended Practice for Software Requirements Specifications

* ISO/IEC 29148:2018 — Systems and software engineering — Requirements engineering

* Martin Fowler — Patterns of Enterprise Application Architecture (2002)

* Chris Richardson — Microservices Patterns (2018)

* Spring Boot 4.x Reference Documentation — https://docs.spring.io/spring-boot

* Apache Kafka Documentation — https://kafka.apache.org/documentation

* Spring Data JPA Reference — https://docs.spring.io/spring-data/jpa/docs/current/reference/html

## **1.5 Overview**

Tài liệu được cấu trúc gồm 9 phần chính: (1) Introduction, (2) Overall Description, (3) System Architecture, (4) Functional Requirements, (5) Non-Functional Requirements, (6) Database Design, (7) External Interface Requirements, (8) Development Roadmap, và (9) Appendix. Mỗi phần cung cấp mức độ chi tiết đủ để nhóm phát triển triển khai và nhóm kiểm thử xây dựng test case.

# **2. Overall Description**

## **2.1 Product Perspective**

VaultX là hệ thống độc lập (greenfield), không kế thừa codebase từ hệ thống nào trước đó. Hệ thống được thiết kế theo kiến trúc microservices, giao tiếp với các thành phần ngoài qua REST API và Kafka. VaultX có thể tích hợp với các provider tỷ giá thực trong tương lai thay thế mock feed hiện tại.

| Context: VaultX hoạt động tương tự các nền tảng như MoMo, ZaloPay, GrabPay về nghiệp vụ — nhưng được xây dựng từ đầu với kiến trúc rõ ràng hơn và component C++ FX Engine để học sâu về hệ thống phân tán. |
| :---- |

> **Lưu ý về chiến lược phát triển:** C++ FX Engine sẽ được phát triển song song với các service Java ngay từ giai đoạn đầu của dự án, thay vì mock tạm thời rồi thay thế sau. Điều này đảm bảo dữ liệu tỷ giá luôn đến từ engine thật, đồng thời giúp nhóm phát triển song song cả hai ngôn ngữ.

## **2.2 Product Functions**

Các nhóm chức năng chính của VaultX:

| Function Group | Key Capabilities |
| ----- | ----- |
| Identity & Auth | Đăng ký, đăng nhập, JWT, OAuth2 (Google), KYC-lite, RBAC (user/premium/admin) |
| Wallet Management | Tạo ví đa tiền tệ, double-entry ledger, xem số dư, lịch sử giao dịch |
| Transfer & FX | P2P transfer, FX conversion, Saga rollback, idempotency, outbox pattern |
| FX Rate Engine | C++17 TCP server (Boost.Asio) nhận mock rate feed, tính spread, publish Kafka; Java Kafka consumer cache vào Redis TTL 10s |
| Spending Analytics | Phân loại chi tiêu, trend theo thời gian, so sánh budget vs actual, export CSV |
| Fraud Detection | Velocity check, unusual amount, new device flag, hold \+ notify |
| Notification | Email, push notification, WebSocket real-time balance update |
| Admin Panel | Quản lý user, KYC review, flagged transaction, system ledger, FX config |

## **2.3 User Classes and Characteristics**

| User Class | Characteristics | Access Level |
| ----- | ----- | ----- |
| Guest | Chưa đăng nhập, chỉ xem landing page và tỷ giá công khai. | Read-only public |
| Registered User | Đã đăng ký và xác thực email. Có thể nạp tiền mock, chuyển tiền nội bộ, xem analytics cơ bản. | Standard |
| Premium User | Đã hoàn thành KYC-lite. Có thể thực hiện FX conversion, nhận rate ưu đãi hơn, xem analytics nâng cao. | Elevated |
| Admin | Nhân viên vận hành hệ thống. Có quyền xem ledger toàn hệ thống, review KYC, cấu hình FX rate, xử lý flagged transaction. | Full admin |

## **2.4 Operating Environment**

* Backend services triển khai trên Kubernetes (GKE hoặc tương đương), tối thiểu 3 node worker.

* Database: PostgreSQL 15+ trên managed service (Cloud SQL hoặc self-hosted).

* Message broker: Apache Kafka 3.x (KRaft mode — không dùng ZooKeeper từ Kafka 3.3+).

* Cache: Redis 7.x standalone (hoặc Redis Cluster cho production-like setup).

* FX Rate Engine: C++17, build với CMake 3.20+, Boost 1.82+, librdkafka 2.x.

* Frontend: React 18, chạy trên CDN hoặc static hosting (Vercel, Netlify, S3 \+ CloudFront).

* CI/CD: GitHub Actions pipeline; container registry Docker Hub hoặc GitHub Container Registry (GHCR).

* Observability: Prometheus \+ Grafana \+ Jaeger distributed tracing.

## **2.5 Design and Implementation Constraints**

* Tất cả Java service sử dụng Spring Boot 4.x, Java 21 LTS. Ba thư viện core bắt buộc: Spring Boot, Spring Data JPA, Spring Security.

* C++ FX Engine sử dụng C++17 standard, không dùng thư viện có license không tương thích.

* Database schema tuân thủ 3NF (Third Normal Form).

* Mọi giao dịch tài chính phải sử dụng double-entry ledger; không được update balance trực tiếp ngoài ledger flow.

* FX conversion chỉ được thực thi sau khi có tỷ giá từ FX Engine (không hardcode tỷ giá).

* Idempotency key bắt buộc cho tất cả transfer request.

* Không lưu trữ thông tin thẻ thanh toán thật (PCI-DSS scope). Schema migration bắt buộc dùng Flyway — không dùng spring.jpa.hibernate.ddl-auto=create/update trong staging/production. Entity mapping giữa layer dùng MapStruct; không dùng manual setter chain hoặc ModelMapper. Distributed lock cho wallet balance dùng Redisson (Redlock algorithm) thay vì DB-level lock thuần.

## **2.6 Assumptions and Dependencies**

* Tỷ giá từ FX Rate Engine là mock feed — trong phiên bản hiện tại chưa kết nối provider thực.

* KYC-lite chỉ yêu cầu upload ảnh CMND; hệ thống chưa tích hợp OCR hoặc eKYC thực tế.

* Nạp tiền (top-up) là mock — hệ thống ghi nhận lệnh nạp nhưng không xử lý thanh toán thực.

* Hệ thống phụ thuộc vào Kafka cho inter-service communication — nếu Kafka down, các saga flow sẽ bị trì hoãn.

* OTP trong flow chuyển tiền là mock 6 chữ số gửi qua email; chưa tích hợp SMS gateway.

* **C++ FX Engine được phát triển đồng thời với các service Java ngay từ đầu, sử dụng mock feed simulator riêng.** Kafka topic `fx.rates.raw` sẽ có dữ liệu từ C++ engine ngay từ Phase 1, thay vì dùng mock consumer Java.

# **3. System Architecture**

(Giữ nguyên toàn bộ kiến trúc như bản 1.1, bao gồm Service Catalogue, Inter-Service Communication, C++ FX Engine Technical Detail, Saga Flow.)

## **3.1 Architecture Overview**

VaultX áp dụng kiến trúc microservices với 6 Java service riêng biệt, mỗi service có database riêng (Database-per-Service pattern). Hệ thống áp dụng kiến trúc đa giao thức (Multi-Protocol Architecture) để tối ưu hoá giao tiếp:
1. **GraphQL (External API):** Giao tiếp giữa Frontend (React) và Backend thông qua một API Gateway / BFF (Backend For Frontend). GraphQL giúp tránh over-fetching dữ liệu cho các màn hình phức tạp.
2. **gRPC (Internal Service-to-Service):** Giao tiếp đồng bộ nội bộ giữa các Java Microservices sử dụng gRPC (Protobuf) để đạt hiệu năng cao và strongly-typed contract.
3. **REST API (Edge Cases):** Được giữ lại cho các luồng đặc thù như OAuth2 (Google Login), Webhook, hoặc Upload file (multipart/form-data).

Giao tiếp bất đồng bộ vẫn được xử lý qua Apache Kafka (KRaft mode). C++ FX Rate Engine là component biệt lập, giao tiếp với Java consumer qua Kafka topic. Mỗi Java service áp dụng layered architecture chuẩn: Controller/gRPC Service → Service → Repository (Spring Data JPA), với DTO↔Entity mapping qua MapStruct và schema versioning qua Flyway.

## **3.2 Service Catalogue**

| Service | Language / Framework | Responsibility | Database |
| ----- | ----- | ----- | ----- |
| GraphQL Gateway / BFF | Spring Boot 4 \+ Spring GraphQL (hoặc Node.js Apollo) | Single GraphQL endpoint cho Frontend, routing queries/mutations tới các gRPC/REST backend services, rate limiting, JWT validation | N/A (Stateless) |
| Identity Service | Spring Boot 4 \+ Spring Security \+ gRPC | Auth, JWT issue/refresh, OAuth2 (REST), KYC upload (REST), RBAC; cung cấp gRPC endpoint cho user info | PostgreSQL; Flyway migration; MapStruct DTO↔Entity |
| Wallet Service | Spring Boot 4 \+ Spring Data JPA \+ gRPC | Multi-currency ledger, double-entry, balance query qua gRPC, tx history; JPA @Lock (PESSIMISTIC\_WRITE); Redisson lock | PostgreSQL; Flyway; Spring Data Redis (balance cache TTL 5s) |
| Transfer Service | Spring Boot 4 \+ Spring Kafka \+ Spring Data JPA \+ gRPC | P2P transfer, FX conversion, Saga choreography qua Kafka; @Transactional boundary rõ ràng; Outbox relay bằng Quartz | PostgreSQL; Flyway; Quartz job store (in-memory hoặc JDBC) |
| Analytics Service | Spring Boot 4 \+ Spring Batch \+ Spring Data JPA \+ gRPC | Spending aggregation, CQRS materialized view, budget alert, CSV export (REST); Kafka consumer cập nhật snapshot | PostgreSQL; Flyway; Spring Batch JobRepository |
| Notification Service | Spring Boot 4 \+ Spring Web MVC \+ Spring Kafka | Kafka consumer; email via JavaMailSender; WebSocket real-time qua STOMP over WebSocket (Spring MVC, không dùng reactive); session mapping qua Redis | Redis (STOMP session store); PostgreSQL (notification\_log) |
| FX Rate Engine | C++17 \+ Boost.Asio | TCP server, rate feed ingestion, spread calculation, Kafka publish | Stateless (Redis cache via Java consumer) |

## **3.3 Inter-Service Communication**

### **3.3.1 Synchronous Communication (GraphQL, gRPC, REST)**

*   **External (Client → BFF):** Client gọi API qua duy nhất một endpoint GraphQL (vd: `/graphql`) tại GraphQL Gateway. Gateway sẽ validate JWT, extract user context, và phân giải (resolve) các GraphQL query/mutation thành các lời gọi tương ứng xuống các backend services.
*   **Internal (Service → Service):** Các backend Java services gọi lẫn nhau thông qua **gRPC** qua HTTP/2. Các file `.proto` định nghĩa contract rõ ràng và được quản lý chung trong một module/thư mục. Circuit breaker pattern được implement bằng Resilience4j tại mỗi gRPC client stub.
*   **REST Edge Cases:** Client vẫn gọi REST trực tiếp (hoặc qua Gateway routing) cho các endpoint như: `/api/v1/auth/oauth2` (redirect flow) hoặc `/api/v1/kyc/upload` (multipart form).

### **3.3.2 Asynchronous Communication (Kafka Topics)**

| Kafka Topic | Producer | Consumer(s) | Purpose |
| ----- | ----- | ----- | ----- |
| fx.rates.raw | FX Rate Engine (C++) | Wallet Service (Java) | Publish tỷ giá raw mỗi 5 giây |
| fx.rates.processed | Wallet Service | Transfer Service, Frontend (via WS) | Tỷ giá đã tính spread, sẵn sàng dùng |
| transfer.initiated | Transfer Service | Wallet Service, Notification Service | Bắt đầu saga chuyển tiền |
| wallet.debited | Wallet Service | Transfer Service | Xác nhận debit thành công trong saga |
| wallet.credited | Wallet Service | Transfer Service, Notification Service | Xác nhận credit thành công |
| transfer.completed | Transfer Service | Analytics Service, Notification Service | Saga hoàn thành — trigger analytics \+ notify |
| transfer.failed | Transfer Service | Wallet Service, Notification Service | Saga thất bại — trigger compensating transaction |
| fraud.flagged | Transfer Service | Notification Service, Admin Service | Phát hiện giao dịch bất thường |
| kyc.submitted | Identity Service | Notification Service | Người dùng submit KYC — notify admin |

## **3.4 C++ FX Rate Engine — Technical Detail**

FX Rate Engine là một TCP server bất đồng bộ xây dựng với Boost.Asio. Server chấp nhận kết nối từ mock rate feed simulator (một process riêng), nhận raw rate data theo binary protocol (Protobuf), tính toán mid/bid/ask rate, rồi publish lên Kafka topic fx.rates.raw mỗi 5 giây.

| Component | Technology | Detail |
| ----- | ----- | ----- |
| TCP Async Server | Boost.Asio (C++17) | Chấp nhận nhiều kết nối đồng thời bằng io\_context \+ strand |
| Rate Feed Simulator | C++17 | Mock process gửi raw rate data qua TCP; thay thế external provider |
| Binary Protocol | Protobuf v3 | Định dạng message: currency\_pair, raw\_mid, timestamp, source\_id |
| Spread Calculation | C++17 arithmetic | bid \= mid \* (1 \- spread\_bps/10000), ask \= mid \* (1 \+ spread\_bps/10000) |
| Kafka Producer | librdkafka (C) | Publish message vào topic fx.rates.raw; acks=all; retry=3 |
| Rate Cache Update | Java Kafka Consumer | Java service consume topic, cache vào Redis với TTL 10s |

## **3.5 Saga Flow — FX Transfer**

Đây là flow phức tạp nhất trong hệ thống. Khi người dùng chuyển 100 USD sang VND, hệ thống thực hiện các bước theo thứ tự sau. Nếu bất kỳ bước nào thất bại, compensating transaction được kích hoạt theo chiều ngược lại.

| Step | Service | Action | On Failure |
| ----- | ----- | ----- | ----- |
| 1 | Transfer Service | Validate request, generate idempotency key, check duplicate | Return 409 Conflict |
| 2 | Transfer Service | Fetch current USD/VND rate từ Redis cache (published bởi FX Engine) | Return 503 Rate Unavailable |
| 3 | Wallet Service | Acquire Redisson distributed lock (key: wallet:{wallet\_id}); JPA @Lock(PESSIMISTIC\_WRITE) tại DB; verify sufficient balance | Release lock, return 422 |
| 4 | Wallet Service | Ghi LedgerEntry (DEBIT) trong @Transactional; cập nhật balance\_after; publish wallet.debited qua Outbox (cùng transaction) | Release lock, compensate step 3 |
| 5 | Transfer Service | Calculate VND amount \= USD \* rate \* (1 \- fee\_rate) | Compensate: credit back USD (step 4 reverse) |
| 6 | Wallet Service | Ghi LedgerEntry (CREDIT) trong @Transactional; cập nhật balance\_after; publish wallet.credited qua Outbox | Compensate: reverse debit (step 4\) |
| 7 | Transfer Service | Mark transfer COMPLETED, publish transfer.completed event | Compensate all previous steps |
| 8 | Analytics Service | Consume transfer.completed, update spending snapshot | Retry via Kafka (non-critical) |
| 9 | Notification Service | Consume transfer.completed, push WebSocket \+ email | Retry via Kafka (non-critical) |

# **4\. Functional Requirements**

## **4.1 Identity Service**

### **FR-ID-01: User Registration**

Người dùng cung cấp email, mật khẩu, và họ tên để tạo tài khoản. Hệ thống gửi email xác thực. Tài khoản chỉ được kích hoạt sau khi người dùng xác nhận email.

| Attribute | Detail |
| ----- | ----- |
| Input | email (unique), password (min 8 ký tự, ít nhất 1 chữ hoa, 1 số, 1 ký tự đặc biệt), full\_name |
| Validation | Email format check, password strength check, email uniqueness check |
| Output | 201 Created \+ user\_id; email xác thực được gửi |
| Error Cases | 400 Bad Request (validation fail), 409 Conflict (email đã tồn tại) |

### **FR-ID-02: User Login (JWT)**

Người dùng đăng nhập bằng email và mật khẩu. Hệ thống trả về access token (JWT, 15 phút) và refresh token (HttpOnly cookie, 7 ngày).

| Attribute | Detail |
| ----- | ----- |
| Input | email, password |
| Output | 200 OK \+ { access\_token, token\_type: Bearer, expires\_in: 900 }; refresh\_token trong HttpOnly cookie |
| Error Cases | 401 Unauthorized (sai credentials), 403 Forbidden (email chưa verify), 429 Too Many Requests (\>5 lần sai trong 5 phút) |

### **FR-ID-03: OAuth2 Login (Google)**

Người dùng có thể đăng nhập qua Google OAuth2. Nếu email chưa tồn tại, hệ thống tự động tạo tài khoản và bỏ qua bước verify email. Nếu email đã tồn tại, hệ thống liên kết với tài khoản hiện có.

### **FR-ID-04: JWT Refresh**

Client dùng refresh token để lấy access token mới mà không cần đăng nhập lại. Refresh token được rotate sau mỗi lần dùng (refresh token rotation).

### **FR-ID-05: KYC-Lite Submission**

Người dùng upload ảnh CMND/CCCD (front \+ back). Hệ thống lưu file, chuyển trạng thái KYC sang PENDING, notify admin. Admin review và cập nhật trạng thái thành APPROVED hoặc REJECTED.

| KYC Status | Meaning |
| ----- | ----- |
| NONE | Người dùng chưa submit KYC — chỉ có quyền Standard |
| PENDING | Đã submit, đang chờ admin review |
| APPROVED | KYC được duyệt — nâng lên Premium user |
| REJECTED | KYC bị từ chối — giữ Standard, có thể submit lại |

## **4.2 Wallet Service**

### **FR-WL-01: Multi-Currency Wallet Creation**

Sau khi đăng ký thành công, hệ thống tự động tạo ví VND mặc định cho người dùng. Người dùng có thể tạo thêm ví cho các loại tiền tệ được hỗ trợ (USD, EUR, JPY, SGD). Mỗi người dùng có tối đa một ví cho mỗi loại tiền tệ.

### **FR-WL-02: Double-Entry Ledger**

Mọi thay đổi số dư đều được ghi qua cặp ledger entry (debit \+ credit). Mỗi giao dịch tạo đúng 2 bản ghi: một bản ghi DEBIT từ tài khoản nguồn, một bản ghi CREDIT vào tài khoản đích. Tổng của tất cả ledger entry trong hệ thống tại bất kỳ thời điểm nào có xu hướng tiệm cận 0\.

| Ledger Invariant: SUM(debit\_amount) \- SUM(credit\_amount) ≈ 0 tại mọi thời điểm. Đây là invariant cơ bản của double-entry accounting. Hệ thống có reconciliation job kiểm tra điều kiện này định kỳ. |
| :---- |

### **FR-WL-03: Balance Query**

Người dùng xem số dư hiện tại của tất cả ví. Số dư được tính bằng: balance \= SUM(credit\_amount) \- SUM(debit\_amount) cho tài khoản đó. Kết quả được cache Redis với TTL ngắn (5 giây) để giảm tải DB.

### **FR-WL-04: Transaction History**

Người dùng xem lịch sử giao dịch với phân trang (page, size), lọc theo loại tiền tệ, khoảng thời gian, và loại giao dịch. Response bao gồm transaction\_id, type, amount, currency, counterparty, created\_at, status.

### **FR-WL-05: Mock Top-Up**

Người dùng có thể yêu cầu nạp tiền mock (không qua payment gateway thực). Hệ thống tạo ledger entry credit vào ví người dùng và debit từ tài khoản hệ thống (SYSTEM\_FLOAT). Giới hạn mock top-up: 10,000,000 VND hoặc tương đương/lần.

## **4.3 Transfer Service**

### **FR-TR-01: P2P Transfer (Same Currency)**

Người dùng chuyển tiền cho người dùng khác trong cùng loại tiền tệ. Flow thực hiện qua Saga pattern với idempotency key.

| Field | Validation Rule |
| ----- | ----- |
| recipient\_email hoặc recipient\_id | Người nhận phải tồn tại và có ví loại tiền tương ứng |
| amount | Lớn hơn 0, không vượt quá available\_balance của sender |
| currency | Phải thuộc danh sách tiền tệ hỗ trợ |
| idempotency\_key | UUID v4, bắt buộc; nếu key đã tồn tại và trạng thái COMPLETED, trả về kết quả cũ |
| note | Tối đa 255 ký tự, tùy chọn |

### **FR-TR-02: FX Conversion Transfer**

Người dùng chuyển đổi tiền giữa hai loại tiền tệ khác nhau. Hệ thống lấy tỷ giá hiện tại từ Redis cache (publish bởi FX Engine), tính toán số tiền nhận, và hiển thị fee \+ rate breakdown để người dùng xác nhận trước khi thực thi.

| Attribute | Detail |
| ----- | ----- |
| Rate Source | Redis cache, key: fx:rate:{from\_currency}:{to\_currency}, TTL 10 giây |
| Fee Structure | Flat fee: 0.5% trên số tiền gốc; tối thiểu 2,000 VND hoặc tương đương |
| Rate Lock | Tỷ giá được lock 30 giây sau khi preview; nếu user xác nhận sau 30 giây, hệ thống lấy lại tỷ giá mới |
| Prerequisite | Chỉ Premium user (đã KYC) có thể thực hiện FX conversion |

### **FR-TR-03: Idempotency**

Mỗi transfer request phải đính kèm idempotency\_key. Hệ thống lưu idempotency\_key vào bảng idempotency\_keys với transfer\_id và status. Nếu cùng key được gửi lại trong vòng 24 giờ, hệ thống trả về response giống với lần xử lý đầu tiên mà không thực hiện lại giao dịch.

### **FR-TR-04: Outbox Pattern**

Khi Transfer Service ghi transfer record, đồng thời ghi outbox event trong cùng một database transaction. Một relay process riêng poll outbox table và publish event lên Kafka. Cơ chế này đảm bảo event không bị mất ngay cả khi Kafka tạm thời không khả dụng.

## **4.4 Analytics Service**

### **FR-AN-01: Spending Categorization**

Mỗi transfer có thể được gán một category (ăn uống, di chuyển, mua sắm, giải trí, y tế, khác). Người dùng có thể tự gán category. Analytics Service aggregate theo category để hiển thị phân bổ chi tiêu.

### **FR-AN-02: Spending Timeline**

Người dùng xem biểu đồ chi tiêu theo ngày hoặc tháng trong khoảng 30 hoặc 90 ngày. Data được tổng hợp từ materialized view (spending\_snapshots) cập nhật theo hai cơ chế: (1) Spring Batch Job chạy mỗi đêm lúc 02:00 UTC, đọc ledger\_entries qua JpaPagingItemReader (chunk size 500), tổng hợp qua ItemProcessor, ghi vào spending\_snapshots qua JpaItemWriter; (2) Kafka consumer cập nhật snapshot incremental sau mỗi transfer.completed event để dashboard có data gần-thời-gian-thực trong ngày.

### **FR-AN-03: Budget vs Actual**

Người dùng thiết lập ngân sách (budget) theo category và tháng. Analytics Service so sánh chi tiêu thực tế với ngân sách. Khi chi tiêu vượt 80% ngân sách của một category, hệ thống gửi budget alert qua Notification Service.

### **FR-AN-04: Export CSV**

Người dùng xuất lịch sử giao dịch theo khoảng thời gian ra file CSV. File bao gồm các cột: date, type, amount, currency, category, counterparty, note, status. Download qua pre-signed URL có hiệu lực 10 phút.

## **4.5 Fraud Detection**

### **FR-FD-01: Velocity Check**

Hệ thống đếm số lần transfer của một user trong cửa sổ trượt 1 phút bằng Redis sliding window counter. Nếu trong 1 phút có nhiều hơn 5 transfer thành công, hệ thống flag giao dịch tiếp theo là SUSPICIOUS và tạm giữ (HOLD).

### **FR-FD-02: Unusual Amount Detection**

Hệ thống tính trung bình 30 ngày của số tiền transfer của user. Nếu một transfer vượt quá 10 lần trung bình này, giao dịch được flag SUSPICIOUS. User nhận thông báo và phải xác nhận lại qua email OTP trong 15 phút, nếu không giao dịch bị huỷ.

### **FR-FD-03: New Device Detection**

Hệ thống lưu device fingerprint (user\_agent \+ IP subnet) của mỗi lần đăng nhập. Khi phát hiện login từ device mới, hệ thống gửi email xác nhận. Transfer từ device mới (chưa xác nhận) trong 24 giờ đầu có thể bị flag tùy cấu hình.

## **4.6 Admin Panel**

### **FR-AD-01: User Management**

Admin xem danh sách người dùng với filter theo trạng thái KYC, ngày đăng ký, và trạng thái tài khoản. Admin có thể lock/unlock tài khoản, và review+approve/reject KYC submission.

### **FR-AD-02: Flagged Transaction Review**

Admin xem danh sách giao dịch bị flag với context đầy đủ (user profile, lịch sử, lý do flag). Admin có thể approve (release HOLD) hoặc reject (cancel giao dịch, trigger compensating transaction).

### **FR-AD-03: System Ledger View**

Admin xem ledger toàn hệ thống với filter theo tài khoản, loại tiền tệ, và khoảng thời gian. Bao gồm view reconciliation report hiển thị tổng debit vs credit để phát hiện bất thường.

### **FR-AD-04: FX Rate Configuration**

Admin cấu hình spread\_bps (basis points) cho từng cặp tiền tệ. Thay đổi có hiệu lực sau khi FX Rate Engine publish batch tỷ giá tiếp theo. Lịch sử thay đổi spread được ghi lại kèm admin\_id và timestamp.

# **5\. Non-Functional Requirements**

## **5.1 Performance**

| Metric | Target | Measurement Method |
| ----- | ----- | ----- |
| API Response Time (P95) | \< 200ms cho read endpoints; \< 500ms cho write endpoints | Prometheus histogram \+ Grafana dashboard |
| Analytics Dashboard Query | \< 50ms với dữ liệu lên đến 1 triệu ledger entry | CQRS materialized view \+ DB index optimization |
| FX Rate Latency | Tỷ giá mới publish lên Kafka trong khoảng 5 giây từ khi rate feed cập nhật | Kafka consumer lag metric |
| WebSocket Balance Update | Balance update đến client trong khoảng 1 giây sau khi credit hoàn tất | End-to-end trace với Jaeger |
| Throughput (Peak) | Hệ thống có khả năng xử lý khoảng 500 concurrent users trong giai đoạn demo/staging | k6 load test |

## **5.2 Availability & Reliability**

| Attribute | Target |
| ----- | ----- |
| Uptime Target | 99.5% trong môi trường staging (không áp dụng cho dev environment) |
| Saga Compensation | Compensating transaction được thực thi trong khoảng 30 giây sau khi phát hiện failure |
| Kafka Retry | Kafka consumer retry với exponential backoff (1s, 2s, 4s, 8s, tối đa 5 lần) |
| Circuit Breaker | Resilience4j circuit breaker mở sau 5 lần failure liên tiếp; nửa mở sau 30 giây |
| Database Migration (Flyway) | Flyway versioned migration (V1\_\_, V2\_\_, ...) với checksum validation; rollback bằng undo script; migration chạy tự động khi service khởi động (spring.flyway.enabled=true) |

## **5.3 Security**

| Security Control | Implementation |
| ----- | ----- |
| Authentication | JWT Bearer token (RS256), access token TTL 15 phút, refresh token TTL 7 ngày |
| Authorization | Spring Security \+ RBAC; mỗi endpoint được annotate rõ role được phép |
| Transport Security | HTTPS/TLS 1.3 cho tất cả external communication; mTLS cho inter-service (Kubernetes) |
| Password Storage | BCrypt với cost factor 12 |
| SQL Injection Prevention | Spring Data JPA parameterized queries; không dùng native SQL string concat |
| Rate Limiting | Nginx ngx\_http\_limit\_req\_module: 100 req/phút cho authenticated, 20 req/phút cho anonymous; burst=10 nodelay. Spring Security thêm @RateLimiter (Resilience4j standalone) tại login, transfer endpoint. |
| Sensitive Data | Số CMND/CCCD được mask trong log; không log JWT token; password không bao giờ được log |
| Audit Log | Tất cả admin action được ghi vào bảng audit\_logs với actor\_id, action, timestamp |

## **5.4 Scalability**

* Horizontal scaling: tất cả Java service là stateless, có thể scale ngang bằng Kubernetes HPA.

* FX Rate Engine có thể chạy nhiều instance; tỷ giá được publish Kafka — consumer tự cân bằng tải.

* Database: read replica có thể được bổ sung cho Analytics Service; Wallet Service dùng connection pool (HikariCP).

* Kafka partition: mỗi topic có tối thiểu 3 partition để cho phép parallel consumption.

## **5.5 Maintainability**

* Mỗi service có Dockerfile riêng và docker-compose entry để chạy standalone.

* API được document bằng OpenAPI 3.0 (Swagger UI), tự động generate từ annotation.

* Distributed tracing với Jaeger: mỗi request có correlation ID (trace\_id) xuyên suốt các service.

* Log format chuẩn JSON với các field: timestamp, level, service, trace\_id, message.

## **5.6 Observability**

| Tool | Purpose | Key Metrics / Features |
| ----- | ----- | ----- |
| Prometheus | Metrics collection | JVM heap, GC pause, HTTP request rate/duration (Micrometer auto-instrument), Kafka consumer lag, Redisson lock wait time, Flyway migration status, active STOMP sessions |
| Grafana | Metrics visualization | Dashboard per-service từ Micrometer metrics; alert khi error rate \> 1% hoặc P95 latency \> 1s; Spring Batch job execution history dashboard |
| Jaeger | Distributed tracing | End-to-end trace cho mỗi request; span cho DB query, Kafka publish/consume, Redis call |
| GitHub Actions | CI/CD pipeline | Build → Unit test → Integration test → Docker build → Push to registry → Deploy to staging |

# **6\. Database Design (3NF)**

## **6.1 Design Principles**

Tất cả bảng trong VaultX tuân thủ Third Normal Form (3NF): (1) mọi attribute phụ thuộc hoàn toàn vào primary key, (2) không có phụ thuộc bắc cầu (transitive dependency). Ngoài ra, thiết kế áp dụng thêm các nguyên tắc sau:

* Tất cả primary key là UUID v4 (type: uuid) để tránh sequential guessing và hỗ trợ distributed generation.

* Tất cả bảng có created\_at và updated\_at (timestamp with time zone).

* Soft delete được áp dụng cho user và wallet (deleted\_at nullable).

* Ledger entry là append-only; không bao giờ update hoặc delete.

* Enum được lưu dạng VARCHAR với CHECK constraint thay vì PostgreSQL ENUM type để dễ migration.

## **6.2 Identity Service Schema**

**Table: users**

| Column Name | Data Type | Nullable | PK / FK | Description |
| ----- | ----- | :---: | :---: | ----- |
| **id** | uuid | NOT NULL | **PK** | Primary key, UUID v4 |
| email | varchar(255) | NOT NULL |  | Email duy nhất, lowercase, dùng để đăng nhập |
| password\_hash | varchar(255) | YES |  | BCrypt hash của mật khẩu; NULL nếu chỉ dùng OAuth2 |
| full\_name | varchar(255) | NOT NULL |  | Họ và tên đầy đủ của người dùng |
| role | varchar(20) | NOT NULL |  | CHECK IN ('USER','PREMIUM','ADMIN'). Mặc định 'USER' |
| email\_verified | boolean | NOT NULL |  | Mặc định FALSE; TRUE sau khi xác thực email |
| status | varchar(20) | NOT NULL |  | CHECK IN ('ACTIVE','LOCKED','SUSPENDED'). Mặc định 'ACTIVE' |
| created\_at | timestamptz | NOT NULL |  | Thời điểm tạo tài khoản (UTC) |
| updated\_at | timestamptz | NOT NULL |  | Thời điểm cập nhật cuối (UTC) |
| deleted\_at | timestamptz | YES |  | NULL nếu chưa xoá; soft delete |

**Table: email\_verifications**

| Column Name | Data Type | Nullable | PK / FK | Description |
| ----- | ----- | :---: | :---: | ----- |
| **id** | uuid | NOT NULL | **PK** | Primary key |
| user\_id | uuid | NOT NULL | **FK** | FK → users.id |
| token | varchar(255) | NOT NULL |  | Token xác thực email (random UUID) |
| expires\_at | timestamptz | NOT NULL |  | Hết hạn sau 24 giờ từ khi tạo |
| used\_at | timestamptz | YES |  | NULL nếu chưa dùng; điền khi token được consume |
| created\_at | timestamptz | NOT NULL |  | Thời điểm tạo |

**Table: refresh\_tokens**

| Column Name | Data Type | Nullable | PK / FK | Description |
| ----- | ----- | :---: | :---: | ----- |
| **id** | uuid | NOT NULL | **PK** | Primary key |
| user\_id | uuid | NOT NULL | **FK** | FK → users.id |
| token\_hash | varchar(255) | NOT NULL |  | SHA-256 hash của refresh token; không lưu token thô |
| device\_fingerprint | varchar(500) | YES |  | User-agent \+ IP subnet hash để detect new device |
| expires\_at | timestamptz | NOT NULL |  | Hết hạn sau 7 ngày |
| revoked\_at | timestamptz | YES |  | NULL nếu còn hiệu lực; điền khi revoke (logout/rotation) |
| created\_at | timestamptz | NOT NULL |  | Thời điểm tạo |

**Table: kyc\_submissions**

| Column Name | Data Type | Nullable | PK / FK | Description |
| ----- | ----- | :---: | :---: | ----- |
| **id** | uuid | NOT NULL | **PK** | Primary key |
| user\_id | uuid | NOT NULL | **FK** | FK → users.id |
| front\_image\_url | varchar(1000) | NOT NULL |  | URL ảnh mặt trước CMND/CCCD (S3 hoặc tương đương) |
| back\_image\_url | varchar(1000) | NOT NULL |  | URL ảnh mặt sau CMND/CCCD |
| status | varchar(20) | NOT NULL |  | CHECK IN ('PENDING','APPROVED','REJECTED') |
| reviewed\_by | uuid | YES | **FK** | FK → users.id (admin); NULL nếu chưa review |
| review\_note | text | YES |  | Ghi chú của admin khi reject |
| submitted\_at | timestamptz | NOT NULL |  | Thời điểm submit |
| reviewed\_at | timestamptz | YES |  | Thời điểm review; NULL nếu chưa review |

**Table: oauth2\_accounts**

| Column Name | Data Type | Nullable | PK / FK | Description |
| ----- | ----- | :---: | :---: | ----- |
| **id** | uuid | NOT NULL | **PK** | Primary key |
| user\_id | uuid | NOT NULL | **FK** | FK → users.id |
| provider | varchar(50) | NOT NULL |  | CHECK IN ('GOOGLE','FACEBOOK'); tên OAuth2 provider |
| provider\_user\_id | varchar(255) | NOT NULL |  | ID của user trên provider (Google sub claim) |
| provider\_email | varchar(255) | NOT NULL |  | Email trên provider account |
| created\_at | timestamptz | NOT NULL |  | Thời điểm liên kết |

## **6.3 Wallet Service Schema**

**Table: currencies**

| Column Name | Data Type | Nullable | PK / FK | Description |
| ----- | ----- | :---: | :---: | ----- |
| **code** | varchar(3) | NOT NULL | **PK** | ISO 4217 currency code: VND, USD, EUR, JPY, SGD |
| name | varchar(100) | NOT NULL |  | Tên đầy đủ: Vietnamese Dong, US Dollar, ... |
| symbol | varchar(10) | NOT NULL |  | Ký hiệu: ₫, $, €, ¥, S$ |
| decimal\_places | smallint | NOT NULL |  | Số chữ số thập phân: VND=0, USD=2, EUR=2 |
| is\_active | boolean | NOT NULL |  | Mặc định TRUE; FALSE nếu ngừng hỗ trợ |

**Table: wallets**

| Column Name | Data Type | Nullable | PK / FK | Description |
| ----- | ----- | :---: | :---: | ----- |
| **id** | uuid | NOT NULL | **PK** | Primary key |
| user\_id | uuid | NOT NULL | **FK** | FK → users.id |
| currency\_code | varchar(3) | NOT NULL | **FK** | FK → currencies.code |
| label | varchar(100) | YES |  | Tên tuỳ chỉnh của ví (vd: 'Tiết kiệm USD') |
| status | varchar(20) | NOT NULL |  | CHECK IN ('ACTIVE','FROZEN','CLOSED'). Mặc định 'ACTIVE' |
| created\_at | timestamptz | NOT NULL |  | Thời điểm tạo ví |
| updated\_at | timestamptz | NOT NULL |  | Cập nhật cuối |
| deleted\_at | timestamptz | YES |  | Soft delete |

Unique constraint: (user\_id, currency\_code) WHERE deleted\_at IS NULL — mỗi user chỉ có một ví active cho mỗi loại tiền tệ.

**Table: ledger\_entries**

| Column Name | Data Type | Nullable | PK / FK | Description |
| ----- | ----- | :---: | :---: | ----- |
| **id** | uuid | NOT NULL | **PK** | Primary key, append-only |
| wallet\_id | uuid | NOT NULL | **FK** | FK → wallets.id |
| transfer\_id | uuid | YES | **FK** | FK → transfers.id; NULL nếu là top-up/system entry |
| entry\_type | varchar(10) | NOT NULL |  | CHECK IN ('DEBIT','CREDIT') |
| amount | numeric(20,6) | NOT NULL |  | Giá trị tuyệt đối của entry; luôn dương |
| currency\_code | varchar(3) | NOT NULL | **FK** | FK → currencies.code |
| balance\_after | numeric(20,6) | NOT NULL |  | Số dư sau khi entry được ghi — dùng cho audit trail |
| description | varchar(500) | YES |  | Mô tả giao dịch (vd: 'P2P Transfer to user@email.com') |
| created\_at | timestamptz | NOT NULL |  | Thời điểm ghi entry (không thay đổi sau khi tạo) |

Index: wallet\_id \+ created\_at DESC (cho transaction history query), transfer\_id (cho reconciliation).

## **6.4 Transfer Service Schema**

**Table: transfers**

| Column Name | Data Type | Nullable | PK / FK | Description |
| ----- | ----- | :---: | :---: | ----- |
| **id** | uuid | NOT NULL | **PK** | Primary key |
| sender\_wallet\_id | uuid | NOT NULL | **FK** | FK → wallets.id |
| receiver\_wallet\_id | uuid | NOT NULL | **FK** | FK → wallets.id |
| transfer\_type | varchar(20) | NOT NULL |  | CHECK IN ('P2P','FX\_CONVERSION','TOP\_UP','FEE') |
| from\_amount | numeric(20,6) | NOT NULL |  | Số tiền gốc từ sender |
| from\_currency | varchar(3) | NOT NULL | **FK** | FK → currencies.code |
| to\_amount | numeric(20,6) | NOT NULL |  | Số tiền nhận tại receiver (sau FX conversion nếu có) |
| to\_currency | varchar(3) | NOT NULL | **FK** | FK → currencies.code |
| fx\_rate | numeric(20,10) | YES |  | Tỷ giá áp dụng; NULL nếu cùng currency |
| fee\_amount | numeric(20,6) | NOT NULL |  | Phí giao dịch; 0 nếu miễn phí |
| fee\_currency | varchar(3) | YES | **FK** | FK → currencies.code; loại tiền tệ thu phí |
| status | varchar(20) | NOT NULL |  | CHECK IN ('PENDING','PROCESSING','COMPLETED','FAILED','CANCELLED','HOLD') |
| idempotency\_key\_id | uuid | NOT NULL | **FK** | FK → idempotency\_keys.id |
| note | varchar(500) | YES |  | Ghi chú của người gửi |
| created\_at | timestamptz | NOT NULL |  | Thời điểm tạo transfer |
| completed\_at | timestamptz | YES |  | Thời điểm hoàn thành; NULL nếu chưa xong |
| failed\_at | timestamptz | YES |  | Thời điểm fail; NULL nếu chưa fail |

**Table: idempotency\_keys**

| Column Name | Data Type | Nullable | PK / FK | Description |
| ----- | ----- | :---: | :---: | ----- |
| **id** | uuid | NOT NULL | **PK** | Primary key |
| key\_value | varchar(255) | NOT NULL |  | Giá trị idempotency key do client cung cấp (UUID v4) |
| user\_id | uuid | NOT NULL | **FK** | FK → users.id |
| transfer\_id | uuid | YES | **FK** | FK → transfers.id; NULL cho đến khi transfer được tạo |
| status | varchar(20) | NOT NULL |  | CHECK IN ('PROCESSING','COMPLETED','FAILED') |
| response\_body | jsonb | YES |  | Response body được cache để trả lại khi duplicate request |
| expires\_at | timestamptz | NOT NULL |  | Hết hạn sau 24 giờ; cron job cleanup |
| created\_at | timestamptz | NOT NULL |  | Thời điểm tạo |

Unique constraint: (key\_value, user\_id) — idempotency key là duy nhất trong phạm vi một user.

**Table: outbox\_events**

| Column Name | Data Type | Nullable | PK / FK | Description |
| ----- | ----- | :---: | :---: | ----- |
| **id** | uuid | NOT NULL | **PK** | Primary key |
| aggregate\_type | varchar(100) | NOT NULL |  | Loại aggregate: 'Transfer', 'Wallet' |
| aggregate\_id | uuid | NOT NULL |  | ID của aggregate (transfer\_id hoặc wallet\_id) |
| event\_type | varchar(100) | NOT NULL |  | Tên event: 'TransferInitiated', 'WalletDebited', ... |
| payload | jsonb | NOT NULL |  | Event payload đầy đủ |
| kafka\_topic | varchar(255) | NOT NULL |  | Kafka topic đích |
| status | varchar(20) | NOT NULL |  | CHECK IN ('PENDING','PUBLISHED','FAILED') |
| published\_at | timestamptz | YES |  | NULL cho đến khi publish thành công |
| retry\_count | smallint | NOT NULL |  | Số lần retry; tối đa 5 |
| created\_at | timestamptz | NOT NULL |  | Thời điểm ghi outbox |

**Table: fraud\_flags**

| Column Name | Data Type | Nullable | PK / FK | Description |
| ----- | ----- | :---: | :---: | ----- |
| **id** | uuid | NOT NULL | **PK** | Primary key |
| transfer\_id | uuid | NOT NULL | **FK** | FK → transfers.id |
| user\_id | uuid | NOT NULL | **FK** | FK → users.id |
| flag\_reason | varchar(50) | NOT NULL |  | CHECK IN ('VELOCITY\_LIMIT','UNUSUAL\_AMOUNT','NEW\_DEVICE','MANUAL') |
| detail | text | YES |  | Chi tiết lý do flag (vd: '6 transfers in 60 seconds') |
| status | varchar(20) | NOT NULL |  | CHECK IN ('OPEN','RESOLVED\_APPROVE','RESOLVED\_REJECT') |
| resolved\_by | uuid | YES | **FK** | FK → users.id (admin); NULL nếu chưa xử lý |
| resolved\_at | timestamptz | YES |  | Thời điểm resolve |
| created\_at | timestamptz | NOT NULL |  | Thời điểm flag |

## **6.5 Analytics Service Schema**

**Table: spending\_categories**

| Column Name | Data Type | Nullable | PK / FK | Description |
| ----- | ----- | :---: | :---: | ----- |
| **id** | uuid | NOT NULL | **PK** | Primary key |
| name | varchar(50) | NOT NULL |  | Tên category: food, transport, shopping, entertainment, health, other |
| icon | varchar(50) | YES |  | Tên icon để render trên frontend |
| is\_system | boolean | NOT NULL |  | TRUE \= category mặc định hệ thống; FALSE \= custom của user |

**Table: transfer\_categories**

| Column Name | Data Type | Nullable | PK / FK | Description |
| ----- | ----- | :---: | :---: | ----- |
| **id** | uuid | NOT NULL | **PK** | Primary key |
| transfer\_id | uuid | NOT NULL | **FK** | FK → transfers.id |
| category\_id | uuid | NOT NULL | **FK** | FK → spending\_categories.id |
| assigned\_by | uuid | NOT NULL | **FK** | FK → users.id — người gán category |
| assigned\_at | timestamptz | NOT NULL |  | Thời điểm gán |

**Table: spending\_snapshots**

| Column Name | Data Type | Nullable | PK / FK | Description |
| ----- | ----- | :---: | :---: | ----- |
| **id** | uuid | NOT NULL | **PK** | Primary key |
| user\_id | uuid | NOT NULL | **FK** | FK → users.id |
| currency\_code | varchar(3) | NOT NULL | **FK** | FK → currencies.code |
| category\_id | uuid | YES | **FK** | FK → spending\_categories.id; NULL \= tổng tất cả category |
| period\_type | varchar(10) | NOT NULL |  | CHECK IN ('DAILY','MONTHLY') |
| period\_date | date | NOT NULL |  | Ngày (cho DAILY) hoặc ngày đầu tháng (cho MONTHLY) |
| total\_spent | numeric(20,6) | NOT NULL |  | Tổng chi tiêu trong kỳ |
| tx\_count | integer | NOT NULL |  | Số giao dịch trong kỳ |
| updated\_at | timestamptz | NOT NULL |  | Lần cập nhật snapshot gần nhất |

Unique constraint: (user\_id, currency\_code, category\_id, period\_type, period\_date).

**Table: budgets**

| Column Name | Data Type | Nullable | PK / FK | Description |
| ----- | ----- | :---: | :---: | ----- |
| **id** | uuid | NOT NULL | **PK** | Primary key |
| user\_id | uuid | NOT NULL | **FK** | FK → users.id |
| category\_id | uuid | NOT NULL | **FK** | FK → spending\_categories.id |
| currency\_code | varchar(3) | NOT NULL | **FK** | FK → currencies.code |
| budget\_amount | numeric(20,6) | NOT NULL |  | Ngân sách cho category này trong tháng |
| month | date | NOT NULL |  | Tháng áp dụng (ngày đầu tháng, vd: 2026-05-01) |
| alert\_threshold | numeric(5,2) | NOT NULL |  | Ngưỡng cảnh báo (0.80 \= 80%); mặc định 0.80 |
| created\_at | timestamptz | NOT NULL |  | Thời điểm tạo |
| updated\_at | timestamptz | NOT NULL |  | Cập nhật cuối |

## **6.6 FX Rate Schema (Wallet Service DB)**

**Table: fx\_rates**

| Column Name | Data Type | Nullable | PK / FK | Description |
| ----- | ----- | :---: | :---: | ----- |
| **id** | uuid | NOT NULL | **PK** | Primary key |
| from\_currency | varchar(3) | NOT NULL | **FK** | FK → currencies.code |
| to\_currency | varchar(3) | NOT NULL | **FK** | FK → currencies.code |
| mid\_rate | numeric(20,10) | NOT NULL |  | Tỷ giá trung tâm từ C++ engine |
| bid\_rate | numeric(20,10) | NOT NULL |  | Tỷ giá mua \= mid \* (1 \- spread\_bps/10000) |
| ask\_rate | numeric(20,10) | NOT NULL |  | Tỷ giá bán \= mid \* (1 \+ spread\_bps/10000) |
| spread\_bps | smallint | NOT NULL |  | Spread tính theo basis points; cấu hình bởi admin |
| source | varchar(50) | NOT NULL |  | CHECK IN ('CPP\_ENGINE','MANUAL'); nguồn cung cấp rate |
| effective\_at | timestamptz | NOT NULL |  | Thời điểm tỷ giá có hiệu lực |
| created\_at | timestamptz | NOT NULL |  | Thời điểm ghi vào DB |

Index: (from\_currency, to\_currency, effective\_at DESC) — query tỷ giá gần nhất cho cặp tiền tệ.

**Table: fx\_rate\_configs**

| Column Name | Data Type | Nullable | PK / FK | Description |
| ----- | ----- | :---: | :---: | ----- |
| **id** | uuid | NOT NULL | **PK** | Primary key |
| from\_currency | varchar(3) | NOT NULL | **FK** | FK → currencies.code |
| to\_currency | varchar(3) | NOT NULL | **FK** | FK → currencies.code |
| spread\_bps | smallint | NOT NULL |  | Spread áp dụng; admin cấu hình qua Admin Panel |
| configured\_by | uuid | NOT NULL | **FK** | FK → users.id (admin) |
| created\_at | timestamptz | NOT NULL |  | Thời điểm cấu hình |

## **6.7 Audit & System Tables**

**Table: audit\_logs**

| Column Name | Data Type | Nullable | PK / FK | Description |
| ----- | ----- | :---: | :---: | ----- |
| **id** | uuid | NOT NULL | **PK** | Primary key |
| actor\_id | uuid | NOT NULL | **FK** | FK → users.id — người thực hiện hành động |
| action | varchar(100) | NOT NULL |  | Tên hành động: 'KYC\_APPROVE', 'FRAUD\_RESOLVE', 'FX\_CONFIG\_UPDATE', ... |
| target\_type | varchar(50) | YES |  | Loại đối tượng bị tác động: 'User', 'Transfer', 'KycSubmission' |
| target\_id | uuid | YES |  | ID của đối tượng bị tác động |
| detail | jsonb | YES |  | Chi tiết bổ sung (before/after state, lý do, ...) |
| ip\_address | inet | YES |  | IP address của actor |
| created\_at | timestamptz | NOT NULL |  | Thời điểm hành động xảy ra |

## **6.8 Key Database Indexes**

| Table | Index Columns | Type | Purpose |
| ----- | ----- | ----- | ----- |
| users | email | UNIQUE | Đăng nhập và duplicate check |
| wallets | (user\_id, currency\_code) WHERE deleted\_at IS NULL | UNIQUE PARTIAL | Đảm bảo 1 ví/currency/user |
| ledger\_entries | (wallet\_id, created\_at DESC) | BTREE | Transaction history query |
| ledger\_entries | transfer\_id | BTREE | Reconciliation lookup |
| transfers | (sender\_wallet\_id, created\_at DESC) | BTREE | Sender history query |
| transfers | (receiver\_wallet\_id, created\_at DESC) | BTREE | Receiver history query |
| transfers | idempotency\_key\_id | UNIQUE | Idempotency lookup |
| idempotency\_keys | (key\_value, user\_id) | UNIQUE | Duplicate request detection |
| outbox\_events | (status, created\_at) WHERE status \= 'PENDING' | PARTIAL BTREE | Relay process polling |
| fx\_rates | (from\_currency, to\_currency, effective\_at DESC) | BTREE | Latest rate lookup |
| spending\_snapshots | (user\_id, period\_type, period\_date DESC) | BTREE | Analytics dashboard query |
| fraud\_flags | (status) WHERE status \= 'OPEN' | PARTIAL BTREE | Admin review queue |
| audit\_logs | (actor\_id, created\_at DESC) | BTREE | Admin activity history |

# **7\. External Interface Requirements**

## **7.1 API Conventions (GraphQL & REST)**

| Convention | Rule |
| ----- | ----- |
| GraphQL Endpoint | `/graphql` tại BFF Gateway. Phục vụ 90% traffic frontend. |
| REST Base URL | `/api/v1/{service}` cho các ngoại lệ (auth, upload). |
| Authentication | Authorization: Bearer {access\_token} (áp dụng cho cả GraphQL HTTP header và REST) |
| Idempotency | Mutation chuyển tiền có tham số `idempotencyKey: UUID!` bắt buộc. |
| Pagination | Relay-style cursor pagination cho GraphQL (connections/edges/nodes). |
| Error Format | GraphQL `errors` array với `extensions` chứa `{ code, detail, traceId }` |

## **7.2 Key GraphQL Definitions & REST Endpoints**

**GraphQL (Chính):**
*   **Queries:**
    *   `me: User!`
    *   `wallets: [Wallet!]!`
    *   `walletTransactions(walletId: ID!, first: Int, after: String): TransactionConnection!`
    *   `fxRatePreview(from: String!, to: String!, amount: Float!): FxPreview!`
    *   `spendingAnalytics(period: String!): SpendingSummary!`
*   **Mutations:**
    *   `register(input: RegisterInput!): AuthPayload!`
    *   `login(input: LoginInput!): AuthPayload!`
    *   `createWallet(currency: String!): Wallet!`
    *   `transfer(input: TransferInput!): TransferPayload!` (Yêu cầu `idempotencyKey`)

**REST API (Ngoại lệ):**
*   `POST /api/v1/auth/refresh` (Dùng HttpOnly cookie)
*   `GET /api/v1/auth/oauth2/google` (Redirect)
*   `POST /api/v1/kyc/submit` (multipart/form-data)
*   `GET /api/v1/analytics/export` (Trả về file CSV)

## **7.3 WebSocket Interface**

Client kết nối WebSocket tại ws://gateway/ws (qua Nginx upgrade). Sau khi handshake, client dùng STOMP protocol để subscribe vào personal channel /user/{user\_id}/queue/notifications. Notification Service (Spring MVC \+ SimpMessagingTemplate) gửi message đến channel này khi có event từ Kafka. STOMP session mapping được lưu vào Redis để hỗ trợ nhiều instance Notification Service (nếu scale ngang).

| Event Type | Trigger | Payload |
| ----- | ----- | ----- |
| BALANCE\_UPDATE | Sau khi credit/debit ledger entry hoàn tất | { wallet\_id, currency, new\_balance, change\_amount, transfer\_id } |
| TRANSFER\_COMPLETED | Sau khi saga transfer hoàn thành | { transfer\_id, status, from\_amount, to\_amount, counterparty\_name } |
| FRAUD\_ALERT | Sau khi fraud flag được tạo | { transfer\_id, reason, action\_required, deadline } |
| BUDGET\_ALERT | Khi chi tiêu vượt threshold | { category\_name, budget\_amount, spent\_amount, percentage, currency } |

## **7.4 C++ FX Engine — Kafka Message Format**

FX Rate Engine publish message lên Kafka topic fx.rates.raw với format Protobuf v3.

| Field | Type | Description |
| ----- | ----- | ----- |
| source\_id | string | ID của rate feed source (vd: 'MOCK\_FEED\_01') |
| from\_currency | string | ISO 4217 code |
| to\_currency | string | ISO 4217 code |
| mid\_rate | double | Tỷ giá trung tâm |
| timestamp\_utc\_ms | int64 | Unix timestamp milliseconds |
| sequence\_num | int64 | Số thứ tự tăng dần để detect out-of-order message |
# **8. Development Roadmap**

## **8.1 Phase Overview (Cập nhật – Tích hợp C++ sớm)**

| Phase | Name | Duration | Demo Milestone |
| ----- | ----- | ----- | ----- |
| 1 | Foundation: Auth, Wallet, P2P Transfer & C++ FX Engine | 5–6 tuần | Đăng ký, đăng nhập, ví đa tiền, P2P same-currency, C++ engine publish tỷ giá |
| 2 | FX Conversion, Saga & Outbox | 4–5 tuần | FX conversion end-to-end với tỷ giá từ C++, Saga rollback, outbox pattern |
| 3 | Analytics, Batch, Fraud Detection & Notification | 4–5 tuần | Dashboard chi tiêu, Spring Batch, fraud alert, WebSocket thời gian thực |
| 4 | KYC & Admin Panel | 3–4 tuần | KYC submission/review, admin quản lý user, fraud flag, cấu hình spread |
| 5 | DevOps & Observability | 3–4 tuần | Kubernetes, CI/CD, Prometheus, Grafana, Jaeger, load test |

## **8.2 Phase 1 — Foundation: Auth, Wallet, P2P & C++ FX Engine (5–6 tuần)**

Mục tiêu: Xây dựng các service nền tảng và có dữ liệu tỷ giá thật từ C++ engine chạy liên tục.

**Deliverables:**

- **C++ FX Rate Engine & Simulator:** TCP server (Boost.Asio) nhận dữ liệu từ mock feed, tính mid/bid/ask, publish lên Kafka topic `fx.rates.raw`. Chạy trong container Docker. C++ simulator gửi rate mỗi 2-5 giây.
- **Java Kafka Consumer** (có thể trong Wallet Service hoặc một service nhỏ `fx-gateway`): Consume `fx.rates.raw`, tính spread theo cấu hình cứng (ví dụ 50 bps), cache vào Redis (`fx:rate:<from>:<to>`, TTL 10s).
- **Identity Service:** Đăng ký, đăng nhập JWT, email verify, refresh token. Tài khoản mặc định role USER.
- **Wallet Service:** Tạo ví VND mặc định khi đăng ký, double-entry ledger, mock top-up, xem số dư (có thể dùng cache Redis), lịch sử giao dịch.
- **Transfer Service (P2P nội tệ):** Chuyển tiền VND → VND đồng bộ với `@Transactional` + Redisson distributed lock + Idempotency key. Chưa có Kafka.
- **API Gateway (BFF):** Spring Boot 4 + Spring GraphQL (cổng 8080). Định nghĩa `schema.graphqls`. JWT validation tại gateway. GraphiQL interface cho dev/testing. Các GraphQL Resolver gọi gRPC xuống Identity, Wallet, Transfer.
- **Docker Compose:** đầy đủ PostgreSQL, Kafka, Redis, Nginx, C++ engine, các Java service.
- **React cơ bản:** Đăng nhập, đăng ký, xem số dư, form chuyển tiền VND.

> **Lưu ý:** Ở phase này, Transfer Service chỉ hỗ trợ chuyển cùng tiền, chưa có FX conversion.

## **8.3 Phase 2 — FX Conversion, Saga & Outbox (4–5 tuần)**

**Deliverables:**

- **Mở rộng Transfer Service** hỗ trợ FX conversion, lấy tỷ giá từ Redis (được cập nhật bởi C++ engine).
- **Saga pattern** cho FX transfer: dùng Kafka với các topic `transfer.initiated`, `wallet.debited`, `wallet.credited`, `transfer.completed`, `transfer.failed`. Outbox pattern trong cả Transfer Service và Wallet Service.
- **Compensating transaction** khi rollback, đảm bảo double-entry ledger cân bằng.
- **Rate lock 30 giây** khi preview, fee calculation.
- **Integration test** mô phỏng các tình huống lỗi.
- **React:** cập nhật giao diện chuyển tiền hiển thị tỷ giá live, fee breakdown.

## **8.4 Phase 3 — Analytics, Batch, Fraud Detection & Notification (4–5 tuần)**

**Deliverables:**

- **Analytics Service:** Spring Batch job đêm, CQRS materialized view, API trả về spending summary và timeline. Kafka consumer cập nhật snapshot sau mỗi `transfer.completed`.
- **Budget tracking:** API quản lý budget, @Scheduled job kiểm tra threshold, gửi cảnh báo sự kiện.
- **Fraud Detection:** Velocity check (sliding window trong Redis), unusual amount detection, new device detection.
- **Notification Service:** Spring WebSocket STOMP gửi BALANCE\_UPDATE, TRANSFER\_COMPLETED, FRAUD\_ALERT. Email notification cho fraud hold và OTP.
- **React:** Dashboard phân tích chi tiêu (Recharts), timeline, budget vs actual, export CSV.

## **8.5 Phase 4 — KYC & Admin Panel (3–4 tuần)**

**Deliverables:**

- **KYC-Lite:** Upload ảnh CMND, admin review flow. KYC approved → nâng role PREMIUM.
- **Admin Panel:** Quản lý user (lock/unlock), danh sách KYC pending, danh sách fraud flag, resolve fraud flag.
- **Admin cấu hình FX:** Giao diện cập nhật spread bps, lưu lịch sử thay đổi.
- **Audit log** cho các hành động admin.

## **8.6 Phase 5 — DevOps & Observability (3–4 tuần)**

**Deliverables:**

- **Kubernetes manifests** cho tất cả service, HPA configuration.
- **CI/CD:** GitHub Actions pipeline (build → test → build Docker image → push registry → deploy).
- **Observability stack:** Prometheus scrape metrics, Grafana dashboard (JVM, Kafka, latency, throughput), Jaeger distributed tracing.
- **Load test:** k6 script với 500 concurrent users, phân tích kết quả.
- **Production-like environment** hoàn chỉnh.

## **8.7 Job Readiness Assessment**

| Milestone | Employability Signal |
| ----- | ----- |
| Phase 1–2 hoàn thành | Microservices thực thụ, C++ engine tích hợp thật, Kafka, Saga – thể hiện năng lực thiết kế hệ thống phân tán và phát triển đa ngôn ngữ |
| Phase 3 hoàn thành | Spring Batch, CQRS, analytics pipeline thời gian thực – phù hợp với phỏng vấn fintech |
| Phase 4–5 hoàn thành | Hệ thống production-ready, K8s, CI/CD, monitoring – đủ sức apply mid/senior backend |

# **9. Appendix**

(Giữ nguyên Technology Evaluation và các phần khác như bản 1.1, cập nhật nếu cần.)

## **9.1 Technology Evaluation — C++ vs Java cho FX Engine**

Việc dùng C++17 cho FX Rate Engine là một lựa chọn có chủ đích để học hỏi, không phải tối ưu đơn thuần. Bảng sau so sánh hai lựa chọn:

| Criterion | C++17 (Chosen) | Java Alternative |
| ----- | ----- | ----- |
| Latency | Latency thấp hơn do ít overhead GC; phù hợp với rate feed liên tục | Spring Web MVC \+ Kafka consumer cũng đạt được throughput tốt với ít overhead hơn; GC pause ít ảnh hưởng hơn ở mức latency 5-giây publish cycle |
| Learning Value | Học được async I/O (Boost.Asio), binary protocol (Protobuf), Kafka C client | Ít learning value hơn vì đã có Java service khác trong project |
| Complexity | Khó debug hơn; build system (CMake) phức tạp hơn Maven/Gradle | Dễ tích hợp hơn vào Spring MVC \+ JPA ecosystem; debug dễ hơn; không cần học Boost.Asio |
| Justification | C++ là điểm khác biệt rõ ràng trong CV; phù hợp với mục tiêu học sâu về systems programming | Đủ cho production nhưng không tạo ra learning differentiation |

## **9.2 Known Limitations & Future Work**

| Limitation | Impact | Potential Future Work |
| ----- | ----- | ----- |
| Mock FX rate feed | Tỷ giá không phản ánh thực tế thị trường | Tích hợp provider thực (Open Exchange Rates, Fixer.io) |
| KYC-lite không có OCR | Admin phải review thủ công, không scalable | Tích hợp OCR service (Tesseract hoặc cloud AI) |
| Mock top-up | Không có dòng tiền thực | Tích hợp VNPay hoặc Stripe cho payment processing |
| Mock OTP qua email | Email OTP chậm và kém UX hơn SMS | Tích hợp SMS gateway (Twilio, ESMS) |
| Single PostgreSQL per service | Có thể gặp bottleneck khi scale ledger\_entries | Partition ledger\_entries theo wallet\_id \+ tháng; hoặc dùng TimescaleDB |
| Fraud detection rule-based | Không bắt được các pattern phức tạp | ML-based anomaly detection (Isolation Forest hoặc Autoencoder) |

## **9.3 Revision History**

| Version | Date | Author | Changes |
| ----- | ----- | ----- | ----- |
| 1.0 | May 2026 | VaultX Engineering Team | Initial SRS — full document, 3NF DB design, all functional & non-functional requirements |
| 1.1 | May 2026 | VaultX Engineering Team | Updated tech stack constraints: Flyway mandatory, MapStruct, Redisson, Spring MVC for WebSocket |
| 1.2 | May 2026 | VaultX Engineering Team | Development roadmap revised: C++ FX Engine developed in parallel from Phase 1; FX conversion moved to Phase 2; overall timeline adjusted for early C++ integration. |