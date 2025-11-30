# 📝 ADR 002: Pattern Timeout (Chống Lỗi Thác Đổ)

**Trạng thái:** Được chấp nhận (Accepted)  
**Lĩnh vực:** Graceful Degradation & Chaos Engineering  

---

## 1. Bối cảnh (Context)

Trong kiến trúc **microservices phân tán**, lỗi chậm trễ (latency) của một service (ví dụ: `UserService` bị chậm do lỗi CSDL) có thể gây **quá tải tài nguyên** và **thác đổ lỗi (cascading failure)** lên service gọi (ví dụ: `TripService`).  
Điều này là **không chấp nhận được** khi traffic dự kiến **tăng gấp 20 lần**.

---

## 2. Quyết định (Decision)

- Áp dụng **Timeout Pattern** (giới hạn thời gian chờ) **1 giây** cho tất cả các lời gọi đồng bộ (`REST/gRPC`) từ các service trung tâm (như `TripService`, `DriverService`) đến các service phụ thuộc.  
- Nếu service phụ thuộc không phản hồi trong 1 giây, trả lỗi **fail fast** để bảo vệ service gọi.

---

## 3. Cân nhắc & Đánh đổi (Trade-offs)

| Tiêu chí          | Phương án 1: Không Timeout | Phương án 2: Áp dụng Timeout (Đã chọn) |
|------------------|---------------------------|---------------------------------------|
| **Ổn định Hệ thống** | Nguy cơ thác đổ lỗi cao, làm sập service gọi | Hệ thống ổn định, cô lập lỗi tại service bị chậm |
| **Trải nghiệm UX**  | Có thể bị treo lâu (ví dụ: 30s) | Trả lỗi nhanh chóng (1 giây), người dùng có thể thử lại ngay (Fail Fast) |
| **Độ phức tạp**    | Thấp hơn | Cao hơn (cần cài đặt & bảo trì thư viện/middleware Timeout) |

---

### Lập luận Kỹ thuật

Chúng tôi **chấp nhận tăng độ phức tạp trong code** để đổi lấy:

- **Khả năng chống chịu (Resilience)** tốt hơn.  
- **Bảo vệ khả năng chịu tải** `~10,000 RPS`.  
- Kiểm chứng bằng các kịch bản **Chaos Engineering**.
