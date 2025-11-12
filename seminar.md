# 🚖 UIT-Go: Nền tảng Đặt Xe Thời Gian Thực theo Kiến trúc Microservice

## 🏗️ 1. Giới thiệu
**UIT-Go** là một dự án xây dựng hệ thống **đặt xe thời gian thực (ride-hailing)** theo mô hình **Microservice Architecture**.  
Mục tiêu của dự án không chỉ là tạo một ứng dụng đặt xe, mà còn **thiết kế một kiến trúc nền tảng vững chắc**, có thể mở rộng, chịu lỗi, và vận hành hiệu quả trong môi trường thực tế quy mô lớn.

---

## 🎯 2. Phạm vi & Phân chia module
Dự án được chia thành **3 nhóm module chính**, tương ứng với 3 vai trò:

| Module | Vai trò | Mục tiêu chính |
|:-------|:---------|:----------------|
| **A. Scalability & Performance (System Architect)** | Thiết kế kiến trúc để đạt hyper-scale, không chỉ tinh chỉnh hệ thống hiện có. |
| **B. Reliability & High Availability (SRE)** | Xây dựng hệ thống có khả năng chống chịu và tự phục hồi khi gặp sự cố. |
| **E. Automation & Cost Optimization (FinOps)** | Thiết kế quy trình tự phục vụ cho developer và tối ưu chi phí vận hành. |

---

## ⚔️ 3. Bài toán thách thức từng module

### 🧱 Module A – Scalability
- Làm sao để **luồng Đặt xe** chịu được tải đột biến (thundering herd)?
- Làm sao để **luồng Cập nhật vị trí** (write-heavy) xử lý hàng chục ngàn request/giây real-time?

### 🛡️ Module B – Reliability
- Làm sao để loại bỏ **Single Point of Failure (SPOF)**?
- Làm sao để **chứng minh hệ thống tự phục hồi** thay vì chỉ hy vọng?

### ⚙️ Module E – FinOps
- Làm sao để developer mới **triển khai service nhanh** mà vẫn an toàn?
- Làm sao để **theo dõi chi phí** và cảnh báo khi vượt ngưỡng?

---

## 💡 4. Giải pháp tiếp cận

### 🔹 Module A – Scalability
**Luồng Đặt xe:**  
- Sử dụng giao tiếp **asynchronous với Kafka**.  
- `TripService` đẩy event vào Kafka, `DriverService` xử lý.  
- **Trade-off:** chấp nhận độ trễ nhỏ để đổi lấy khả năng chịu tải gần như vô hạn.

**Luồng Cập nhật vị trí:**  
- Dùng **Redis GEO** + **gRPC (Protobuf)**.  
- **Trade-off:** tốn RAM hơn nhưng có tốc độ cực nhanh.  
- Đã thử nghiệm **Load Testing** & **Tuning (Caching, Read Replica)**.

---

### 🔹 Module B – Reliability
- **Multi-AZ Deployment:** loại bỏ SPOF cho ALB, ECS, RDS.  
- **Chaos Engineering:** dùng **AWS FIS** để "tiêm lỗi" và kiểm chứng tính tự phục hồi.  
- **Disaster Recovery (DR):** dùng **RDS Cross-Region Snapshot** + **Terraform IaC** để khôi phục toàn bộ hệ thống nhanh chóng.

> 💬 *Trade-off:* Multi-AZ tốn chi phí gấp đôi và tăng latency ghi nhẹ, nhưng đổi lại RTO ≈ 0 (thay vì 15–30 phút của Single-AZ).

---

### 🔹 Module E – FinOps
- **CI/CD tự phục vụ:** GitHub Actions + Terraform Modules tái sử dụng.  
- **Quản lý chi phí:** Tagging tài nguyên bắt buộc, AWS Cost Explorer + Budgets.  
- **Tối ưu compute:** dùng **Graviton ARM** hoặc **Spot Instances** để tiết kiệm chi phí.

---

## 🗺️ 5. Sơ đồ kiến trúc hệ thống
📌 (Sơ đồ minh họa trong file `README.md` — `Image/BASIC.png`)

Sơ đồ thể hiện các thành phần:
- **Services:** Auth, Trip, Driver  
- **Hạ tầng:** Gateway, Service Discovery  
- **Dữ liệu:** PostgreSQL, MongoDB, Redis, Kafka

---

## 🔄 6. Luồng nghiệp vụ chính

### 🚕 Đặt xe (Asynchronous Flow)
1. User gửi yêu cầu đến `TripService`.  
2. `TripService` tạo bản ghi `Trip(PENDING)` trong MongoDB và đẩy `CreateTripEvent` vào Kafka.  
3. `DriverService` đọc event, tìm tài xế gần nhất qua Redis GEO và phát `AcceptTripEvent`.  
4. `TripService` nhận event và cập nhật cho người dùng.

### 📡 Cập nhật vị trí tài xế (Write-heavy)
1. App tài xế mở kênh **gRPC streaming** đến `DriverService`.  
2. Gửi vị trí mỗi khi di chuyển >20m hoặc >5s.  
3. `DriverService` cập nhật vị trí vào Redis qua `GEOADD`.

### 🔐 Auth Service
- Dùng PostgreSQL lưu người dùng.  
- Sinh **JWT Access/Refresh Token** để xác thực với API Gateway.

---

## 🗄️ 7. Chiến lược Database per Service

| Service | Database | Ưu tiên | Trade-off |
|----------|-----------|----------|------------|
| **Auth** | PostgreSQL | Consistency | Bảo toàn dữ liệu người dùng (ACID). |
| **Trip** | MongoDB | Scalability | Linh hoạt cấu trúc, chấp nhận eventual consistency. |
| **Driver** | Redis GEO | Performance | RAM đắt nhưng cần cho truy vấn real-time. |

---

## 🛠️ 8. Công nghệ sử dụng

| Thành phần | Công nghệ |
|-------------|------------|
| **Backend** | Java Spring Boot |
| **Giao tiếp** | REST, WebSocket, gRPC |
| **Hàng đợi** | Kafka |
| **Databases** | PostgreSQL, MongoDB, Redis |
| **Đóng gói** | Docker, Docker Compose |
| **Cloud** | AWS (EC2, ECR, VPC, IAM) |
| **Hạ tầng (IaC)** | Terraform |
| **Kiểm thử** | k6 / JMeter, AWS FIS |

---

## 🚀 9. Triển khai & Vận hành

- **CI/CD:** GitHub Actions tự động test, build, push image lên ECR và deploy qua Terraform.  
- **IaC:** Toàn bộ hạ tầng được định nghĩa bằng Terraform modules tái sử dụng.  
- **High Availability:** Deploy Multi-AZ trên AWS (ALB, ECS, RDS).  
- **Cost Management:** Tagging tài nguyên và giám sát qua AWS Budgets.

---
## 10. Demo unit test và test coverage.
