# ADR 002: Lựa Chọn Cơ Chế Định Vị Microservices
**Trạng thái:** Đã chấp nhận

---

## 📌 Bối cảnh
Khi các **microservice** được triển khai và mở rộng ngang (horizontal scaling), **địa chỉ IP/port thay đổi động**.  
Cần một cơ chế để các service có thể **tìm thấy nhau** mà không cần cấu hình tĩnh (hardcoded endpoint).

---

## 🧩 Quyết định
- **Triển khai Discovery Service** để quản lý và định vị động các service.
- Công nghệ gợi ý: **AWS Cloud Map** hoặc **HashiCorp Consul**.

---

## 🔍 Các lựa chọn đã cân nhắc
1. **Cấu hình tĩnh qua file**  
   - Nhược điểm: Không linh hoạt, cần cập nhật thủ công khi thay đổi service
2. **Sử dụng Load Balancer để phân phối traffic**  
   - Nhược điểm: Không giải quyết vấn đề định vị service trực tiếp, phụ thuộc vào LB

---

## ⚖️ Đánh đổi (Trade-offs)

### ✅ Lợi ích
- Loại bỏ sự phụ thuộc vào **cấu hình tĩnh**
- Hệ thống **linh hoạt**, tự phục hồi, dễ dàng mở rộng ngang

### ⚠️ Chi phí
- Tăng độ phức tạp khi thiết lập và vận hành một **service hạ tầng mới**
- Cần giám sát và bảo trì Discovery Service để đảm bảo độ ổn định

---

## 📝 Kết luận
Việc sử dụng **Discovery Service** giúp các microservice trong UIT-Go **tự định vị lẫn nhau**, đảm bảo khả năng mở rộng ngang và khả năng tự phục hồi, giảm rủi ro từ các endpoint thay đổi động.

