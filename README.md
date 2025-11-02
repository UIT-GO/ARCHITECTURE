# 🏗️ UIT-Go System Architecture

Tài liệu này trình bày **kiến trúc hệ thống tổng quan** và **kiến trúc chi tiết cho các module** của dự án **UIT-Go** — hệ thống đặt xe thời gian thực theo mô hình Microservice.

---

## 🚀 1. Kiến trúc Tổng quan (Giai đoạn 1: “Bộ Xương”)

Giai đoạn này tập trung xây dựng nền tảng **core system** gồm 3 microservices cơ bản và các thành phần hạ tầng tối thiểu.

---

### 📊 1.1 Sơ đồ Kiến trúc

![Architecture Diagram](Image/BASIC.png)
Sơ đồ thể hiện:
- API GATEWAY, Discovery Service
- UserService
- TripService
- DriverService
- Kafka (hoặc SQS, RabbitMQ) cho giao tiếp sự kiện
- Redis / PostgreSQL / MongoDB làm backend cho từng service
- Các giao tiếp sử dụng: RestAPI, HTTP/HTTPS, WEBSOCKET

---

### 🧩 1.2 Mô tả Thành phần

## 🧭 API Gateway

**API Gateway** là **điểm vào duy nhất (entry point)** của toàn bộ hệ thống microservices.  
Tất cả request từ client (mobile app, web app) đều **đi qua Gateway** trước khi đến các service nội bộ như `auth-service`, `trip-service`, `driver-service`, ...

---

### ⚙️ Chức năng chính
- 🔀 **Routing:** Định tuyến request đến đúng microservice tương ứng.  
- 🔒 **Authentication & Authorization:** Kiểm tra token và phân quyền truy cập.  
- 📊 **Rate Limiting & Logging:** Giới hạn tần suất truy cập, ghi log tập trung.  
- 🧩 **Response Aggregation:** Tổng hợp dữ liệu từ nhiều service để giảm số lượng request client cần gửi.  
- 🛡️ **Security Layer:** Che giấu cấu trúc hệ thống nội bộ, tăng cường bảo mật.

---

### 🎯 Vai trò trong kiến trúc
- Đơn giản hóa giao tiếp giữa client và hệ thống backend.  
- Tăng **tính bảo mật**, **dễ quản lý**, và **dễ mở rộng** khi thêm service mới.  
- Hỗ trợ **load balancing**, **caching** và **fallback** khi có service gặp sự cố.

---

### 🚀 Triển khai
| Môi trường | Công nghệ sử dụng |
|-------------|------------------|
| 🏭 **Production** | AWS Application Load Balancer (ALB) hoặc AWS API Gateway |
| 💻 **Development** | Nginx Gateway hoặc Spring Cloud Gateway |

---
## 🔎 Discovery Service

**Discovery Service** chịu trách nhiệm **quản lý và định vị động (dynamic discovery)** các microservices trong hệ thống.  
Thay vì phải cấu hình thủ công địa chỉ IP hoặc hostname, các service sẽ **đăng ký (register)** và **tra cứu (discover)** lẫn nhau thông qua Discovery Service.

---

### ⚙️ Chức năng chính
- 🧭 **Service Registration:** Khi một microservice khởi động, nó tự động đăng ký thông tin (tên service, địa chỉ, cổng) vào Discovery Service.  
- 📡 **Service Lookup:** Các service khác có thể truy vấn để lấy thông tin endpoint hiện tại của service mục tiêu.  
- 🔁 **Dynamic Scaling:** Khi service scale-out (thêm instance mới), Discovery Service tự động cập nhật danh sách node.  
- 💥 **Health Check:** Theo dõi tình trạng hoạt động (health status) của từng instance và loại bỏ các node hỏng.

---

### 🎯 Vai trò trong kiến trúc
- Loại bỏ sự phụ thuộc vào **cấu hình tĩnh (hardcoded endpoint)**.  
- Giúp hệ thống **linh hoạt, tự phục hồi**, và dễ dàng **mở rộng ngang (horizontal scaling)**.  
- Cung cấp nền tảng cho các cơ chế **load balancing thông minh** tại API Gateway hoặc giữa các service với nhau.

---

### 🚀 Triển khai
| Môi trường | Công nghệ sử dụng |
|-------------|------------------|
| 🏭 **Production** | AWS Cloud Map hoặc HashiCorp Consul |
| 💻 **Development** | Netflix Eureka (Spring Cloud Netflix) hoặc Consul local mode |

---
#### 👤 UserService
- **Trách nhiệm:**  
  - Quản lý thông tin người dùng (hành khách & tài xế).  
  - Xử lý đăng ký, đăng nhập, xác thực và quản lý hồ sơ.  
- **Cơ sở dữ liệu:** PostgreSQL (hoặc MySQL).  
- **Kết nối:** Cung cấp REST API cho app client, và gRPC nội bộ cho TripService.  

---

#### 🚗 DriverService
- **Trách nhiệm:**  
  - Theo dõi vị trí tài xế theo thời gian thực.  
  - Quản lý trạng thái (online/offline, đang rảnh, đang chở khách).  
  - Cung cấp API tìm kiếm tài xế gần nhất theo vị trí (5km).  
- **Cơ sở dữ liệu:**  
  - Redis (ElastiCache) với **GEO Commands** — ưu tiên tốc độ.  
  - Hoặc DynamoDB với **Geohash** — ưu tiên khả năng mở rộng.  
- **Event Listener:** Nhận event `CreateTripEvent` từ Kafka, tìm tài xế phù hợp, phát `AcceptTripEvent`.

---

#### 🧳 TripService
- **Trách nhiệm:**  
  - Dịch vụ trung tâm, quản lý toàn bộ vòng đời chuyến đi.  
  - Xử lý logic tạo chuyến (`CreateTripEvent`), gán tài xế, cập nhật trạng thái (`ACCEPTED`, `ON_TRIP`, `COMPLETED`, ...).  
- **Cơ sở dữ liệu:** PostgreSQL hoặc MongoDB.  
- **Event Publisher:** Gửi sự kiện `TripCreated` đến DriverService qua Kafka.

---

### ⚙️ 1.3 Nguyên tắc Thiết kế

#### 🛰️ Giao tiếp giữa các Service
- **gRPC** → Ưu tiên hiệu năng, tốc độ.  
- **RESTful API** → Ưu tiên đơn giản, dễ debug.  

#### 🗄️ Database per Service
- Mỗi service sở hữu và quản lý **database riêng biệt**.  
- Không chia sẻ bảng — tránh coupling giữa các domain.

#### 🐳 Containerization
- Tất cả các service được đóng gói bằng **Docker**.  
- **Docker Compose** dùng để chạy hệ thống local.

#### 🧱 Infrastructure as Code (IaC)
- Toàn bộ hạ tầng (VPC, DB, IAM, ALB...) được mô tả và quản lý bằng **Terraform**.

#### ☁️ Triển khai
- Triển khai container trên AWS:
  - **ECS/EKS** → Ưu tiên giảm công vận hành.  
  - **EC2** → Ưu tiên linh hoạt.  
- Hỗ trợ CI/CD với **GitHub Actions** hoặc **AWS CodePipeline**.

---

### 🧠 1.4 Event Flow (Tóm tắt)
1. **User App** gửi yêu cầu đặt xe → `TripService`.
2. `TripService` tạo bản ghi `Trip` (status: `PENDING`) và gửi event `CreateTripEvent` → Kafka.
3. `DriverService` nhận event, tìm tài xế gần nhất, phát `AcceptTripEvent`.
4. `TripService` nhận `AcceptTripEvent`, cập nhật `Trip` → `ASSIGNED` và gửi thông báo đến user + driver.
5. Khi chuyến đi hoàn tất, `DriverService` phát `CompleteTripEvent`.

---

### 📦 1.5 Công nghệ Sử dụng

| Thành phần | Công nghệ |
|-------------|------------|
| Backend | Java Spring Boot |
| Communication | REST / gRPC |
| Messaging | Kafka |
| Databases | PostgreSQL, MongoDB, Redis |
| Container | Docker, Docker Compose |
| Cloud | AWS (ECS/EKS, ALB, ElastiCache, RDS) |
| IaC | Terraform |
| Monitoring | Prometheus / Grafana |

---

📘 **Tác giả:** UIT-Go Team  
📅 **Phiên bản:** Giai đoạn 1 – “Bộ Xương”  
🧱 **Trạng thái:** Đang triển khai nền tảng core

