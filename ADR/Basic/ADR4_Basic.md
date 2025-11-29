# ADR 004: Lựa Chọn Cơ Chế Giao Tiếp Nội Bộ Chính
**Trạng thái:** Đã chấp nhận

---

## 📌 Bối cảnh
Hệ thống cần xử lý hai loại giao tiếp chính:  
1. **Các lệnh đồng bộ** (ví dụ: lấy hồ sơ người dùng).  
2. **Các sự kiện cốt lõi** cần khả năng mở rộng và chịu tải đột biến (ví dụ: tạo chuyến đi).

---

## 🧩 Quyết định
Sử dụng mô hình **giao tiếp kết hợp**:

- **API/Lệnh Đồng bộ:**  
  - Sử dụng **gRPC** cho giao tiếp nội bộ giữa các services (ưu tiên hiệu năng).  
  - Sử dụng **RESTful API** cho giao tiếp từ Client đến API Gateway/TripService.

- **Sự kiện Bất đồng bộ:**  
  - Sử dụng **Apache Kafka** làm **Message Broker (Event Bus)** cho giao tiếp sự kiện quan trọng (ví dụ: TripService phát sự kiện `trip_created` đến DriverService).

---

## 🔍 Các lựa chọn đã cân nhắc
- Chỉ dùng **RESTful API**: đơn giản nhưng khó chịu tải cao, không tối ưu cho event-driven.  
- Chỉ dùng **Kafka**: hiệu quả với event, nhưng không phù hợp cho các lệnh cần trả về kết quả tức thì.

---

## ⚖️ Đánh đổi (Trade-offs)

### ✅ Lợi ích
- **Kafka:** Hệ thống chống lại các đợt tăng traffic đột biến, tăng khả năng mở rộng và chịu lỗi.  
- **gRPC:** Đảm bảo tốc độ cao cho các lệnh nội bộ giữa services.

### ⚠️ Chi phí
- Tăng **độ phức tạp khi debug** luồng nghiệp vụ bất đồng bộ.  
- Cần vận hành thêm **hạ tầng Kafka/Zookeeper**, tăng chi phí vận hành và quản lý.

---

## 📝 Kết luận
Mô hình kết hợp **gRPC + Kafka** giúp UIT-Go vừa đáp ứng các lệnh nội bộ tốc độ cao, vừa xử lý tốt các sự kiện cường độ cao, đảm bảo **scalability** và **resilience**, mặc dù chi phí quản lý hạ tầng tăng.
