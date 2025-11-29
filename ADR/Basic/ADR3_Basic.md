# ADR 003: Lựa Chọn Công Nghệ Lưu Trữ Dữ Liệu Cho Các Services
**Trạng thái:** Đã chấp nhận

---

## 📌 Bối cảnh
Tuân thủ nguyên tắc **"Database per Service"**, mỗi service cần một loại CSDL **tối ưu cho nghiệp vụ** của nó.

---

## 🧩 Quyết định
- **Auth Service:** Sử dụng **PostgreSQL** để lưu trữ thông tin Người dùng, Vai trò (Roles), và Phân quyền (Permissions).
- **Driver Service & Trip Service:** Sử dụng **MongoDB** để lưu trữ Hồ sơ Tài xế, Cuốc xe, và Vị trí (chính).
- **Caching / GeoStore:** Sử dụng **Redis** cho session, blacklist JWT, và lưu trữ vị trí tài xế tạm thời (GeoStore).

---

## 🔍 Các lựa chọn đã cân nhắc
- Chỉ dùng một loại CSDL (ví dụ: chỉ dùng PostgreSQL cho tất cả các service)  
  - Nhược điểm: Không tối ưu hiệu năng cho các luồng dữ liệu lớn hoặc dữ liệu phi cấu trúc.

---

## ⚖️ Đánh đổi (Trade-offs)

### ✅ Lợi ích
- **Tối ưu hiệu năng và tính toàn vẹn dữ liệu** cho từng service:
  - PostgreSQL: đảm bảo **ACID** cho dữ liệu nhạy cảm (User/Auth)
  - MongoDB: linh hoạt, mở rộng ngang tốt cho dữ liệu phát sinh nhanh (Trip, Driver Profile)
  - Redis: tốc độ truy xuất cao, hỗ trợ caching và GeoStore

### ⚠️ Chi phí
- Tăng **chi phí vận hành** và độ phức tạp quản lý
- Cần duy trì và sao lưu ba loại công nghệ CSDL khác nhau

---

## 📝 Kết luận
Việc lựa chọn **đa công nghệ lưu trữ** giúp hệ thống UIT-Go tối ưu hiệu năng, đảm bảo độ tin cậy và khả năng mở rộng, mặc dù chi phí vận hành và quản lý cao hơn so với giải pháp dùng duy nhất một CSDL.
