# 📝 ADR 001: CSDL Multi-AZ so với Single-AZ (HA & Chi phí)

**Trạng thái:** Được chấp nhận (Accepted)  
**Lĩnh vực:** High Availability & Cost  

---

## 1. Bối cảnh (Context)

UIT-Go đang trong giai đoạn **"Scale or Die"** và downtime gây thiệt hại **$5,000/giờ**.  
CSDL (RDS PostgreSQL) là thành phần **cốt lõi** và là **SPOF lớn nhất** nếu chỉ đặt trong một Availability Zone (AZ).

---

## 2. Quyết định (Decision)

- Triển khai tất cả các CSDL **RDS (PostgreSQL/MySQL)** ở chế độ **Multi-AZ** trên AWS.
- Multi-AZ đảm bảo tự động chuyển đổi dự phòng nếu một AZ gặp sự cố, giảm downtime gần bằng **0**.

---

## 3. Cân nhắc & Đánh đổi (Trade-offs)

| Tiêu chí          | Phương án 1: Single-AZ (Giá rẻ) | Phương án 2: Multi-AZ (Đã chọn) |
|------------------|---------------------------------|--------------------------------|
| **Chi phí**       | Thấp hơn **50%**                | Tăng gấp đôi |
| **RTO (Sau sự cố AZ)** | Cao (15–30 phút)            | Gần như bằng 0 (Tự động chuyển đổi dự phòng) |
| **Độ tin cậy**    | Thấp (SPOF nếu AZ gặp lỗi)      | Rất cao (~99.99% Uptime) |

---

### Lập luận Kinh doanh

Chúng tôi chấp nhận **tăng gấp đôi chi phí** để đảm bảo tính **sẵn sàng của dịch vụ đặt xe**, đây là yêu cầu nghiệp vụ **cốt lõi**.  
Việc đạt **RTO gần như bằng 0** giúp bảo vệ công ty khỏi **thiệt hại $5,000/giờ** do downtime.
