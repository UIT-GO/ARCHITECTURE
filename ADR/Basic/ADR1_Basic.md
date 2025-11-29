# ADR 001: Lựa Chọn Kiến Trúc Tổng Thể

## 🎯 Tên Quyết định
Áp dụng Kiến trúc **Microservices** với nguyên tắc **"Database per Service"**.

---

## 📌 Bối cảnh
Đồ án yêu cầu xây dựng nền tảng cho ứng dụng gọi xe **UIT-Go** với các mục tiêu:

- Khả năng mở rộng (**Scalability**)  
- Độ tin cậy (**Reliability**)  
- Độc lập trong phát triển và triển khai (**Independence**)  

Do đó kiến trúc tổng thể phải hỗ trợ việc phát triển linh hoạt, dễ bảo trì và có thể mở rộng theo nhu cầu thực tế.

---

## 🧩 Quyết định
- Chọn kiến trúc **Microservices**.  
- Xây dựng **3 microservices cốt lõi**:
  - `UserService`
  - `TripService`
  - `DriverService`
- Mỗi service sở hữu cơ sở dữ liệu riêng (**Database per Service**), không chia sẻ chung DB vật lý.

---

## 🔍 Các lựa chọn đã cân nhắc
- **Monolithic Architecture**  
  - Ưu điểm: dễ triển khai, đơn giản cho nhóm nhỏ.  
  - Nhược điểm: khó mở rộng, dễ bottleneck khi ứng dụng lớn.

---

## ⚖️ Đánh đổi (Trade-offs)

### ✅ Lợi ích
- Mở rộng từng service độc lập.
- Cho phép dùng công nghệ phù hợp từng module (polyglot persistence).
- Tăng độ ổn định hệ thống — service hỏng không làm sập toàn bộ.
- Dễ triển khai CI/CD riêng từng service.

### ⚠️ Chi phí
- Tăng độ phức tạp vận hành (orchestration, networking).
- Khó khăn trong quản lý **giao dịch phân tán** (distributed transaction).
- Tốn công xây dựng cơ chế giao tiếp giữa các service (REST/gRPC/message broker).
- Tăng chi phí hạ tầng do nhiều container/pod.

---

## 📝 Kết luận
Kiến trúc Microservices phù hợp với định hướng dài hạn của dự án UIT-Go, đặc biệt về khả năng mở rộng và phát triển độc lập, dù phải đánh đổi bằng độ phức tạp trong giai đoạn vận hành ban đầu.
