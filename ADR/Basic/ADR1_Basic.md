# ADR 001: Lựa Chọn Giao Tiếp Nội Bộ Giữa Các Microservices
**Trạng thái:** Đã chấp nhận

---

## 📌 Bối cảnh
Các service trong hệ thống **UIT-Go** (như `UserService`, `TripService`, `DriverService`) cần giao tiếp với nhau một cách **hiệu quả**, **ổn định**, và **nhanh** để xử lý các luồng nghiệp vụ.  
Do đó, cần đưa ra quyết định về **giao thức giao tiếp chuẩn** giữa các microservices trong nội bộ backend.

---

## 🧩 Quyết định
- **Sử dụng gRPC cho giao tiếp nội bộ giữa các microservices.**
- **Sử dụng RESTful API cho các API công khai (Client-Facing API).**

Cách kết hợp này giúp mỗi giao thức được dùng đúng mục đích:
- gRPC → giao tiếp nội bộ tốc độ cao  
- REST → giao tiếp với client/web/mobile đơn giản và phổ biến

---

## 🔍 Các lựa chọn đã cân nhắc
- **Hoàn toàn dùng RESTful API**  
  - Dễ phát triển và debug  
  - Nhưng tốc độ chậm hơn do HTTP/1.1 và JSON nặng  

- **Hoàn toàn dùng gRPC**  
  - Rất nhanh, tối ưu nhờ HTTP/2 và Protobuf  
  - Nhưng client/mobile khó tương thích

---

## ⚖️ Đánh đổi (Trade-offs)

### ✅ Lợi ích khi dùng gRPC nội bộ
- **Hiệu năng cao**  
  - Nhờ HTTP/2 (multiplexing) + Protobuf (nhẹ, nhị phân)
- **Hợp lý với microservices** cần gọi chéo nhiều và yêu cầu latency thấp
- **Hỗ trợ strongly-typed API** (dễ phát triển và tránh sai sót)

### ⚠️ Chi phí
- **Tăng độ phức tạp phát triển**  
  - Phải định nghĩa file `.proto`
- **Debugging khó hơn** so với REST  
  - Cần tool chuyên dụng để xem request/response
- **Triển khai thêm hệ thống codegen** cho nhiều ngôn ngữ (nếu cần)

---

## 📝 Kết luận
Kết hợp **gRPC nội bộ + REST bên ngoài** là lựa chọn tối ưu cho hệ thống UIT-Go, đảm bảo:
- Hiệu năng cao giữa microservices  
- Tính tương thích và đơn giản cho client  
- Đảm bảo phát triển linh hoạt và dễ mở rộng trong tương lai
