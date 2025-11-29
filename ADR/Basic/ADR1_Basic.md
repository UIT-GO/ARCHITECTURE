# ADR 001: Lựa Chọn Cấu Trúc Giao Tiếp Ngoại Vi (Client-Facing)
**Trạng thái:** Đã chấp nhận

---

## 📌 Bối cảnh
Cần một **điểm vào thống nhất** cho toàn bộ hệ thống Microservices từ client (mobile, web app).  
Các yêu cầu chính:
- Quản lý tập trung các tác vụ xuyên suốt (cross-cutting concerns) như **Authentication**, **Authorization**, **Logging**.
- **Định tuyến** request đến các service nội bộ một cách hiệu quả và an toàn.

---

## 🧩 Quyết định
- **Triển khai API Gateway** làm **entry point duy nhất** cho tất cả các request từ client.
- Sản phẩm sử dụng: **AWS Application Load Balancer (ALB)** hoặc **AWS API Gateway**.

---

## 🔍 Các lựa chọn đã cân nhắc
1. **Cho phép client gọi trực tiếp từng microservice**  
   - Nhược điểm: tốn kém, thiếu an toàn, khó quản lý
2. **Sử dụng một Reverse Proxy đơn giản**  
   - Nhược điểm: thiếu chức năng Authentication/Authorization, Load Balancing

---

## ⚖️ Đánh đổi (Trade-offs)

### ✅ Lợi ích
- **Tăng tính bảo mật**: che giấu cấu trúc nội bộ của các service
- **Dễ quản lý**: tập trung kiểm soát traffic, authentication, authorization
- **Mở rộng dễ dàng**: thêm các cross-cutting concerns mà không ảnh hưởng service nội bộ

### ⚠️ Chi phí
- **Tăng nhẹ latency** do request phải đi qua thêm một hop (Gateway)
- **Chi phí vận hành cao hơn** vì phải duy trì thêm một thành phần hạ tầng

---

## 📝 Kết luận
Việc triển khai **API Gateway** giúp UIT-Go có điểm vào tập trung, bảo mật, dễ quản lý, đồng thời hỗ trợ mở rộng các tính năng xuyên suốt mà không tác động trực tiếp đến các microservice nội bộ.
