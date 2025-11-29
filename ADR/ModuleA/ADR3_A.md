# 📄 Architecture Decision Record (ADR)

**ADR 003: Tối ưu Quản lý Kết nối Database (Connection Pooling)**  
**Trạng thái:** Đã chấp nhận (Accepted)  
**Ngày:** [Ngày hiện tại]  
**Người ra quyết định:** [Tên của bạn – Kỹ sư Kiến trúc Hệ thống]

---

## 1. Bối cảnh (Context)

Trong giai đoạn hệ thống UIT-Go phải xử lý tải cao (ví dụ: 1000 VUs trong load testing), mỗi request tạo một kết nối mới đến Database (PostgreSQL, MongoDB) dẫn đến:

- Chi phí I/O cao cho thao tác **connect → auth → handshake → close**.  
- Làm tăng đáng kể **P95/P99 latency** ngay cả trước khi thực hiện truy vấn.  
- Lãng phí chu kỳ CPU cho việc mở/đóng kết nối liên tục.  

Vấn đề này đặc biệt nghiêm trọng trên microservices như **AuthService**, vốn xử lý lưu lượng lớn và truy cập DB liên tục.

---

## 2. Vấn đề (Problem)

- Việc tạo/hủy kết nối Database cho mỗi request dẫn đến **overhead cao**, làm tăng **P99 Latency**, đặc biệt trong các đợt spike traffic.  
- Hệ thống không thể scale ổn định nếu không tối ưu connection lifecycle.  
- Overhead này tích luỹ gây nghẽn trên DB khi số lượng connections tăng đột biến.

---

## 3. Quyết định (Decision)

**→ Áp dụng Connection Pooling cho tất cả microservices sử dụng RDBMS.**

Cụ thể:

- Sử dụng **HikariCP** (pool mặc định của Spring Boot).  
- Cấu hình Pool Size tối ưu dựa trên tài nguyên thực tế:
  - `maximumPoolSize = 20–50`
  - `connectionTimeout = 30000ms`
  - `idleTimeout = 600000ms`
  - `maxLifetime = 1800000ms`
- Áp dụng trực tiếp cho:
  - AuthService (PostgreSQL)
  - TripService / PaymentService hoặc bất kỳ service nào truy cập RDBMS.

Việc này đảm bảo các kết nối được **tái sử dụng** thay vì tạo mới.

---

## 4. Các lựa chọn đã xem xét (Options Considered)

### **A. Sử dụng Connection Pooling (Được chọn)**

**Ưu điểm**
- Giảm mạnh overhead kết nối.  
- Giảm **P99 latency**, cải thiện throughput.  
- Tối ưu cho traffic cao và kiến trúc microservices.  

**Nhược điểm**
- Yêu cầu cấu hình phù hợp (pool quá nhỏ → dễ nghẽn).  
- Cần giám sát metrics như: active connections, pool exhaustion.

---

### **B. Không dùng Connection Pooling (Rejected)**

**Ưu điểm**
- Dễ triển khai, không cần cấu hình.  

**Nhược điểm**
- Mỗi request tạo/hủy kết nối → overhead cực lớn.  
- P99 latency tăng mạnh, dễ gây timeout.  
- Không phù hợp cho kiến trúc hyperscale / cloud-native.  

---

## 5. Hệ quả (Consequences / Trade-offs)

### **Tác động Tích cực**
- Giảm đáng kể chi phí tạo kết nối.  
- Cho phép hệ thống:
  - Duy trì **P95 latency < 80ms** trong load test.  
  - Tăng throughput lên mức ổn định.  
- Tránh tình trạng “connection storm” khi traffic tăng đột biến.

### **Tác động Tiêu cực**
- Nếu pool size quá nhỏ → thread starvation, service bị chặn khi chờ connection.  
- Nếu pool size quá lớn → quá nhiều connections gây áp lực lên DB.  
- Cần theo dõi thêm quan sát (observability):  
  - Metrics: `Hikari.Active`, `Hikari.Idle`, `Hikari.Pending`.

---

## 6. Liên quan Module A/B

- Đây là kỹ thuật quan trọng trong **Module A – Service Performance Optimization**.  
- Đồng thời cũng góp phần đảm bảo **Reliability & HA** trong **Module B**, vì giảm tình trạng nghẽn DB dẫn đến service outage.

---

## 7. Quyết định cuối cùng

**→ Chấp nhận Connection Pooling với HikariCP làm chuẩn cho mọi microservice có kết nối RDBMS.**  
Cấu hình được lưu trong `application-prod.yml` và được tự động scale theo số lượng vCPU.


