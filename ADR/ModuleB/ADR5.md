# 📝 ADR 003: Chiến lược Disaster Recovery (DR)

**Trạng thái:** Được chấp nhận (Accepted)  
**Lĩnh vực:** Disaster Recovery & Cost Optimization  

---

## 1. Bối cảnh (Context)

- Lãnh đạo công ty yêu cầu kế hoạch **DR** với:  
  - **RTO ≤ 30 phút**  
  - **RPO ≤ 5 phút**  

- Công ty cần **tối ưu chi phí** để nguồn vốn (runway) không cạn kiệt.

---

## 2. Quyết định (Decision)

- Sử dụng chiến lược **Pilot Light** làm nền tảng cho việc phục hồi **liên vùng (Multi-Region DR)**.  
- Chỉ duy trì các thành phần cốt lõi và dữ liệu cần thiết, các thành phần khác được bật khi có sự cố.

---

## 3. Cân nhắc & Đánh đổi (Trade-offs)

| Tiêu chí          | P3: Warm Standby (RTO 5-10min) | P2: Pilot Light (RTO 10-30min) (Đã chọn) | P1: Backup & Restore (RTO Hours) |
|------------------|--------------------------------|-----------------------------------------|---------------------------------|
| **Đáp ứng RTO**   | Có (≤10 phút)                  | Có (≤30 phút)                            | Không |
| **Chi phí**       | Cao                            | Trung bình (Chỉ giữ các thành phần cốt lõi & Data Replication) | Thấp |
| **RPO**           | Minutes                        | Minutes (Đạt RPO 5 phút)                | Hours |

---

### Lập luận Kinh doanh

- **Pilot Light** mang lại **cân bằng tốt nhất** giữa tốc độ phục hồi và chi phí vận hành.  
- Đáp ứng **RTO ≤ 30 phút** và **RPO ≤ 5 phút**.  
- Giữ chi phí DR ở mức hợp lý trong giai đoạn **Startup**, giúp công ty duy trì khả năng phục hồi mà không vượt ngân sách.
