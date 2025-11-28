# 📄 Architecture Decision Record (ADR) 001: Decoupling Luồng Đặt Xe bằng Giao tiếp Bất đồng bộ

**Thuộc tính**

| Thuộc tính             | Mô tả |
|------------------------|-------|
| **Tiêu đề**            | Sử dụng Message Queue (SQS/Kafka) để chuyển đổi luồng đặt xe sang Bất đồng bộ (Asynchronous) |
| **Trạng thái**         | Đã Chấp nhận (Accepted) |
| **Ngày**               | [Ngày hiện tại] |
| **Người ra quyết định**| [Tên của bạn - Kỹ sư Kiến trúc Hệ thống] |

---

## 1. Bối cảnh (Context)

Hệ thống đặt xe hiện tại (Legacy) sử dụng giao tiếp **đồng bộ (Synchronous HTTP)** giữa TripService và DriverService.

- **Vấn đề Latency:** TripService bị blocking trong khi chờ DriverService tìm tài xế và chờ tài xế xác nhận (quá trình có thể kéo dài vài giây).  
- **Vấn đề Tải:** Dưới tải cao (Flash Sale "Đồng giá 5k"), điều này làm cạn kiệt các connection của TripService, dẫn đến:  
  - P95 Latency tăng vọt lên 2,300 ms  
  - Tỷ lệ lỗi (Error Rate) lên đến 18%  
  - Gây mất doanh thu nghiêm trọng (~\$100/phút)

---

## 2. Quyết định (Decision)

Áp dụng mô hình **Message Queue** (AWS SQS hoặc Apache Kafka) để tách rời (decouple) luồng xử lý đặt xe giữa TripService và DriverService.

- TripService sẽ tạo đơn hàng với trạng thái **PENDING** và ngay lập tức gửi Event (ví dụ: `TripCreatedEvent`) vào Queue.  
- TripService trả về cho người dùng **HTTP 202 Accepted** trong vòng < 100ms.  
- DriverService (Consumer) sẽ nhận Event từ Queue và thực hiện logic tìm/gán tài xế.

---

## 3. Cân nhắc (Options Considered)

| Phương án                      | Ưu điểm | Nhược điểm |
|--------------------------------|---------|------------|
| **A. Giữ HTTP Đồng bộ**         | Đơn giản, dễ debug, đảm bảo tính nhất quán tức thời (Immediate Consistency) | Không thể Scale, Latency cao, Error Rate cao, dễ bị sập dưới tải |
| **B. Message Queue (Đã chọn)** | Khả năng chịu tải Hyper-scale, giải phóng tài nguyên tức thời | Tính nhất quán cuối cùng (Eventual Consistency), phức tạp hơn khi debug (cần Correlation ID) |
| **C. Tăng cường HTTP Connection Pool** | Giảm độ trễ một chút | Không giải quyết được bản chất blocking của API |

---

## 4. Hệ quả (Consequences / Trade-offs)

| Loại       | Chi tiết |
|------------|---------|
| **Tích cực (Benefits)** | **Scalability & Độ ổn định:** Max Throughput tăng từ 55 req/s lên 1,250 req/s (gấp 22 lần) <br> **Trải nghiệm người dùng:** P95 Latency giảm từ 2,300 ms xuống 48 ms (gần 47 lần) <br> **Bảo vệ Doanh thu:** Tỷ lệ lỗi giảm từ 18% xuống 0% |
| **Tiêu cực (Drawbacks)** | **Tính nhất quán:** Chấp nhận Eventual Consistency (người dùng phải chờ phản hồi kết quả sau khi tài xế chấp nhận) <br> **Hoạt động (Operations):** Cần thiết lập Monitoring và Tracing (Correlation ID) cho Message Queue để theo dõi luồng xử lý |
