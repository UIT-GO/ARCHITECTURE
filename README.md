# 🏗️ UIT-Go System Architecture

Tài liệu này trình bày **kiến trúc hệ thống tổng quan** và **kiến trúc chi tiết cho các module** của dự án **UIT-Go** — hệ thống đặt xe thời gian thực theo mô hình Microservice.  

Use case triển khai có trong folder `Image`.


---

## 1. Giới thiệu

### 1.2 Giai đoạn 1: “Bộ Xương”

🚀 Giai đoạn này tập trung xây dựng nền tảng **core system** gồm 3 microservices cơ bản và các thành phần hạ tầng tối thiểu.

---

## 2. Kiến trúc tổng quan

### 2.1 Sơ đồ kiến trúc

Sơ đồ thể hiện:  

- API Gateway, Discovery Service  
- Auth Service  
- Trip Service  
- Driver Service  
- Kafka (hoặc SQS, RabbitMQ) cho giao tiếp sự kiện  
- Redis / PostgreSQL / MongoDB làm backend cho từng service  

**Các giao tiếp sử dụng:** REST API, HTTP/HTTPS, WebSocket  

---

## 3. Microservices chính

### 3.1 🧭 API Gateway

**API Gateway** là điểm vào duy nhất (entry point) của toàn bộ hệ thống microservices.  

**Chức năng chính:**

- 🔀 **Routing**: Định tuyến request đến đúng microservice.  
- 🔒 **Authentication & Authorization**: Kiểm tra token và phân quyền truy cập.  
- 🛡️ **Security Layer**: Che giấu cấu trúc hệ thống nội bộ, tăng cường bảo mật.  

**Vai trò trong kiến trúc:**

- Đơn giản hóa giao tiếp client ↔ backend  
- Tăng bảo mật, dễ quản lý, mở rộng  
- Hỗ trợ load balancing, caching và fallback  

**Triển khai:**

| Môi trường | Công nghệ sử dụng |
|------------|-----------------|
| Production | AWS ALB hoặc AWS API Gateway |
| Development | Nginx Gateway hoặc Spring Cloud Gateway |

---

### 3.2 🔎 Discovery Service

**Discovery Service** chịu trách nhiệm quản lý và định vị động các microservices.  

**Chức năng chính:**

- 🧭 **Service Registration**: Tự động đăng ký thông tin service khi khởi động  
- 📡 **Service Lookup**: Truy vấn thông tin endpoint hiện tại  
- 🔁 **Dynamic Scaling**: Cập nhật danh sách node khi scale-out  

**Vai trò trong kiến trúc:**

- Loại bỏ phụ thuộc vào hardcoded endpoint  
- Hỗ trợ tự phục hồi và mở rộng ngang  
- Nền tảng cho load balancing thông minh  

**Triển khai:**

| Môi trường | Công nghệ sử dụng |
|------------|-----------------|
| Production | AWS Cloud Map hoặc HashiCorp Consul |
| Development | Netflix Eureka (Spring Cloud Netflix) hoặc Consul local mode |

---

### 3.3 👤 Auth Service

**Auth Service** quản lý thông tin người dùng (hành khách, tài xế).  

**Chức năng chính:**

- 📝 **Đăng ký (Sign Up)**  
- 🔐 **Đăng nhập (Sign In)**: Sinh JWT Access Token & Refresh Token, lưu Redis  
- ♻️ **Làm mới token**  
- 🧾 **Quản lý hồ sơ**  
- 🧭 **Phân quyền** (ROLE_USER, ROLE_DRIVER)  
- 💬 **Cung cấp thông tin cho các service khác**  

**Kiến trúc & Cơ sở dữ liệu:** PostgreSQL (hoặc MySQL), ORM: JPA/Hibernate  

---

### 3.4 🚖 Driver Service

**Driver Service** quản lý tài xế, vị trí thời gian thực, xử lý sự kiện liên quan cuốc xe.  

**Chức năng chính:**

- 📋 **Quản lý thông tin tài xế**  
- 🛰️ **Theo dõi vị trí thời gian thực** (Redis Geospatial)  
- 🧭 **Tìm kiếm tài xế gần nhất** (bán kính 5km)  
- 💬 **Lắng nghe & phản hồi sự kiện** (Kafka Event)  

**Kiến trúc & Thành phần:**

| Thành phần | Mô tả |
|------------|-------|
| Ngôn ngữ | Java (Spring Boot) |
| Database chính | MongoDB |
| Cache/GeoStore | Redis (Geospatial) |
| Message Broker | Kafka |
| API | RESTful + Event-driven |
| Triển khai | Docker container, giao tiếp qua service discovery |

---

### 3.5 🚘 Trip Service

**Trip Service** quản lý vòng đời cuốc xe.  

**Chức năng chính:**

- 📍 **Tạo cuốc xe mới**  
- 🔄 **Phát event "trip_created" đến Kafka**  
- 🚕 **Nhận event "trip_accepted" từ DriverService**  
- 🧾 **Quản lý vòng đời chuyến đi** (PENDING → ACCEPTED → ONGOING → COMPLETED / CANCELED)  
- 💬 **Gửi thông báo cập nhật trạng thái**  

**Kiến trúc & Thành phần:**

| Thành phần | Mô tả |
|------------|-------|
| Ngôn ngữ | Java (Spring Boot) |
| Database | MongoDB |
| Message Broker | Kafka |
| API | RESTful API (Client ↔ TripService) |
| Triển khai | Docker container |
| Giao tiếp nội bộ | Kafka Event Bus |

---

---

# Infrastructure Architecture (AWS)

### Terraform Infrastructure as Code
Located in `IaC/terraform/` directory with the following structure:

#### Core Infrastructure Components:

**1. Container Registry (ECR)**
- Three ECR repositories for service images:
  - `auth-service`
  - `driver-service` 
  - `trip-service`

**2. Compute Resources**
- **EC2 Instance**: t3.micro (cost-optimized)
- **AMI**: Region-specific (configurable)
- **Instance Profile**: IAM role with ECR read permissions
- **Key Pair**: SSH access for administration

**3. Networking**
- **VPC**: Configurable existing VPC
- **Security Group**: 
  - Inbound: Ports 3030-3032 (service ports)
  - Outbound: All traffic allowed
- **Subnet**: Configurable public subnet

**4. IAM Security**
- **EC2 Instance Role**: ECR read-only access
- **Instance Profile**: Attached to EC2 for container registry access

#### Terraform Configuration Files:

**main.tf**: Core infrastructure resources
**variables.tf**: Configurable parameters
**outputs.tf**: Resource outputs for integration
**terraform.tfvars**: Environment-specific values

### Deployment Automation

**User Data Script** (`user_data.sh`):
- Docker and Docker Compose installation
- AWS CLI setup
- ECR authentication
- Automated service deployment
- Logging and error handling
- Service health monitoring

## Data Architecture

### Database Design

#### 1. PostgreSQL (Auth Service)
- **Database**: `auth_service_db`
- **Tables**: Users, roles, permissions
- **Features**: ACID compliance, relational integrity
- **Port**: 5432

#### 2. MongoDB (Driver & Trip Services)
- **Databases**: `driver-db`, `trip-db`
- **Collections**: Drivers, trips, locations, bookings
- **Features**: Document-based, horizontal scaling
- **Port**: 27017
- **Authentication**: admin/admin123

#### 3. Redis (Caching Layer)
- **Purpose**: Session storage, JWT token blacklisting, temporary data
- **Configuration**: Password-protected (123456)
- **Port**: 6379

### Message Queue Architecture

#### Apache Kafka
- **Purpose**: Event-driven communication between services
- **Port**: 29092 (internal), 9092 (external)
- **Zookeeper**: Coordination service (Port 2181)

**Event Flow**:
```
Auth Service → Kafka → [Driver Service, Trip Service]
Driver Service → Kafka → [Trip Service, Auth Service]
Trip Service → Kafka → [Driver Service, Auth Service]
```

## Containerization & Orchestration

### Docker Configuration

**Individual Dockerfiles**:
- Each service has its own optimized Dockerfile
- Multi-stage builds for reduced image size
- Java 17 runtime environment

**Docker Compose** (`IaC/docker-compose.yml`):
```yaml
services:
  auth-service: Port 3030
  driver-service: Port 3031
  trip-service: Port 3032
  postgres: Port 5432
  mongodb: Port 27017
  redis: Port 6379
  kafka: Port 29092
  zookeeper: Port 2181
```

### Container Registry (ECR)
- Automated image builds and pushes
- Version tagging for rollbacks
- Regional repositories for performance

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
| Communication | REST, WEBSOCKET |
| Messaging | Kafka |
| Databases | PostgreSQL, MongoDB, Redis |
| Container | Docker, Docker Compose |
| Cloud | AWS (EC2/ECR, VPC, AAM) |
| IaC | Terraform |

---
# Testing Strategy
##TEST COVERAGE AUTHSERVICE
![AuthService](Image/testAuthService.jpg)
##TEST COVERAGE TRIPSERVICE
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/abdbe0b0-a916-4025-9f90-6e3d726f8134" />
##TEST COVERAGE DRIVERSERVICE
<img width="1906" height="1029" alt="image" src="https://github.com/user-attachments/assets/62fa77ae-5dfc-49ef-92be-2eda35b94d46" />


---
📘 **Tác giả:** UIT-Go Team  
📅 **Phiên bản:** Giai đoạn 1 – “Bộ Xương”  
🧱 **Trạng thái:** Đang triển khai nền tảng core

