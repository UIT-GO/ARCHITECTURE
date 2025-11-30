# UIT-Go System Architecture Document
**Project:** UIT-Go - Cloud-Native Ride Hailing Platform  
**Team:** [Tên Nhóm của bạn]  
**Modules:** A (Scalability), B (Reliability), E (Automation & Cost)  
**Version:** 1.0.0  

---

## 1. Tổng quan Kiến trúc Hệ thống (System Architecture Overview)

Hệ thống UIT-Go được xây dựng theo kiến trúc **Microservices Event-Driven**, tối ưu cho việc xử lý dữ liệu thời gian thực và mở rộng linh hoạt trên AWS.

---

### 1.1. Sơ đồ Logic (Logical Architecture)

![Architecture Diagram](Image/achitecture.jpg)

#### **Mô tả luồng dữ liệu**
**Giao thức giao tiếp:**

- **gRPC Streaming**  
  Dùng cho luồng cập nhật vị trí thời gian thực giữa Driver App / Customer App và Gateway / DriverService, đảm bảo độ trễ thấp nhất.

- **REST API**  
  Dùng cho nghiệp vụ quản lý (User Profile, Booking History) qua HTTP/HTTPS.

- **Message Broker (Kafka)**  
  Đóng vai trò backbone cho giao tiếp bất đồng bộ, xử lý các sự kiện đặt chuyến, cập nhật trạng thái.

- **Service Discovery**  
  Các microservices tự động tìm thấy nhau khi hạ tầng mở rộng hoặc thay đổi.

---

### 1.2. Sơ đồ Hạ tầng (Infrastructure Architecture)

![Architecture Diagram](Image/overview.jpg)

#### **Mô tả hạ tầng**

- **Multi-AZ Deployment**  
  Hệ thống chạy trên 2 Availability Zones (ap-southeast-1a, ap-southeast-1b) trong 1 VPC (10.10.0.0/16).

- **Load Balancing**  
  Application Load Balancer (ALB) phân phối tải + health check `/health`.

- **Compute Layer**  
  Tất cả Microservices, Database, Kafka, Redis đều chạy dạng container trên EC2 Instances phân bố đều giữa các AZ.

---

## 2. Phân tích Module Chuyên sâu (Deep Dive Modules)

---

### 2.1. Module A — Scalability & Performance

**Mục tiêu:** Xử lý hàng nghìn request vị trí đồng thời và tối ưu hóa luồng dữ liệu.

#### 🔹 Real-time Location Tracking
- Sử dụng **gRPC Streaming** (thay REST) để giảm overhead HTTP và tối ưu băng thông cho GPS updates mỗi 1–3 giây.

#### 🔹 Redis Caching
- Hỗ trợ DriverService và UserService giảm tải query xuống Database chính.

#### 🔹 Event Streaming với Kafka
- Kafka xử lý luồng dữ liệu thông lượng cao, đảm bảo không mất dữ liệu khi tải tăng đột biến (back-pressure).

#### 🔹 Database Strategy – Polyglot Persistence
- **PostgreSQL**: Lưu User, đảm bảo ACID.  
- **MongoDB**: Lưu Trip, Driver attributes, Trip logs — tốc độ ghi cao, schema linh hoạt.

---

### 2.2. Module B — Reliability & High Availability

**Mục tiêu:** Hệ thống không gián đoạn kể cả khi có sự cố hạ tầng.

#### 🔹 High Availability (HA)
- **Active-Active Multi-AZ**: Khi AZ1 lỗi, toàn bộ traffic chuyển sang AZ2.

#### 🔹 Database Replication
- MongoDB + PostgreSQL đều có Replica ở AZ khác → RPO thấp.

#### 🔹 Resilience Patterns
- **Health Checks** qua `/health`.  
- **Auto-Failover** khi EC2 node bị down.  

---

### 2.3. Module E — Automation & Cost Optimization

**Mục tiêu:** Giảm chi phí, tăng automation hạ tầng.

#### 🔹 Cloudwatch:
- Thu log từ microservices và visualize.  

#### 🔹 Infrastructure as Code — Terraform
> *(Giải thích các file Terraform bạn đã/sẽ làm)*  
Tự động hóa dựng VPC, subnet, security group, EC2… đảm bảo Dev/Prod đồng nhất.

---

## 3. Các Quyết định Thiết kế & Đánh đổi (Design Decisions & Trade-offs)

| Quyết định | Lựa chọn | Lý do chọn (Rationale) | Đánh đổi (Trade-off) |
|-----------|----------|------------------------|-----------------------|
| Giao thức vị trí | **gRPC Streaming** | Độ trễ thấp, hiệu năng cao | Debug khó hơn REST, hạn chế cho Browser |
| Message Broker | **Kafka** | Thông lượng cực cao, bền bỉ | Setup phức tạp hơn RabbitMQ/SQS |
| Database Driver | **MongoDB** | Schema linh hoạt, tốc độ ghi cao | Không mạnh ACID như SQL |
| Logging | **ELK Stack** | Search mạnh, chuẩn công nghiệp | ElasticSearch tốn CPU/RAM |

---

## 4. Thách thức & Bài học Kinh nghiệm

### 🔥 Thách thức
- Kafka Cluster trên Docker/EC2 cần nhiều cấu hình tuning (partition, replication factor).
- Xử lý Real-time Streaming (10k+ RPS) cần tối ưu CPU và network.

### 🎯 Bài học
- **Observability cực kỳ quan trọng**.  
  ELK Stack giúp trace theo chiều dọc nhiều microservices thay vì SSH từng server.

---

## 5. Kết quả & Hướng phát triển

### ✅ Kết quả đạt được
- Multi-AZ chạy ổn định.
- Kafka + gRPC Streaming hoạt động mượt ở tải cao.
- Kiến trúc đọc/ghi tách biệt, hiệu năng tốt.

### 🚀 Hướng phát triển
- Chuyển Database sang **Private Subnet** để tăng bảo mật.
- Dùng **Auto Scaling Group (ASG)** cho EC2 để scale tự động theo tải.
- Áp dụng **Serverless (Lambda)** cho một số tác vụ event nhỏ để giảm chi phí.
- Triển khai **Service Mesh** (Istio/Linkerd) để chuẩn hóa quan sát & network policy.


