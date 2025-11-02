# 🏗️ UIT-Go System Architecture

Tài liệu này trình bày **kiến trúc hệ thống tổng quan** và **kiến trúc chi tiết cho các module** của dự án **UIT-Go** — hệ thống đặt xe thời gian thực theo mô hình Microservice.

---

## 🚀 1. Kiến trúc Tổng quan (Giai đoạn 1: “Bộ Xương”)

Giai đoạn này tập trung xây dựng nền tảng **core system** gồm 3 microservices cơ bản và các thành phần hạ tầng tối thiểu.

---

### 📊 1.1 Sơ đồ Kiến trúc

![Architecture Diagram](image/basic.png)
Sơ đồ cần thể hiện:
- Application Load Balancer (ALB)
- UserService
- TripService
- DriverService
- Kafka (hoặc SQS) cho giao tiếp sự kiện
- Redis / PostgreSQL / MongoDB làm backend cho từng service

---

### 🧩 1.2 Mô tả Thành phần

#### 🧭 Application Load Balancer (ALB)
- **Chức năng:** Điểm vào (entry point) của toàn hệ thống.  
- **Vai trò:** Phân phối request HTTP/gRPC đến các service nội bộ tương ứng.  
- **Triển khai:** AWS ALB hoặc Nginx Gateway (ở môi trường dev).

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

