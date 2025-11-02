# 📄 Báo cáo Kế hoạch Chi tiết: Module B (Reliability & High Availability)

**Tên Nhóm:** [Tên nhóm của bạn]  
**Thành viên:** [Tên thành viên 1], [Tên thành viên 2], ...  
**Module đã chọn:** Module B: Thiết kế cho Reliability & High Availability  
**Vai trò đảm nhận:** Kỹ sư Đảm bảo Độ tin cậy (Site Reliability Engineer - SRE)  

---

## 3. Mục tiêu tổng quan
Trình bày kế hoạch chi tiết để **thiết kế một hệ thống có khả năng chống chịu và tự phục hồi**.  
Kế hoạch này tập trung vào 3 nhiệm vụ chính:  
1. Phân tích Điểm lỗi (SPOF)  
2. Thực hành Chaos Engineering  
3. Thiết kế Kịch bản Phục hồi sau Thảm họa (DR)

---

## 1. Lộ trình Thực hiện (Timeline Giai đoạn 2)

| Tuần       | Nhiệm vụ chính                       | Sản phẩm đầu ra (Deliverables) |
|-----------|-------------------------------------|--------------------------------|
| Tuần 9-10 | Nhiệm vụ 1: Phân tích & Loại bỏ Điểm lỗi (HA) | - Sơ đồ kiến trúc Multi-AZ (cập nhật ARCHITECTURE.MD) [cite: 130] <br> - Cấu hình Terraform cho Multi-AZ (ALB, ECS, RDS) <br> - File ADR phân tích Trade-off (Single-AZ vs Multi-AZ) [cite: 131, 87] |
| Tuần 11   | Nhiệm vụ 2: Thực hành Chaos Engineering | - Kịch bản test trên AWS Fault Injection Simulator (FIS) <br> - Log/video chứng minh hệ thống tự phục hồi <br> - (Nếu có) Code hiện thực hóa pattern Retry/Circuit Breaker |
| Tuần 12   | Nhiệm vụ 3: Thiết kế & Thực hành DR | - Tài liệu Kế hoạch DR (tính toán RTO/RPO) <br> - Terraform script cho Region dự phòng (sao lưu) <br> - Log/video thực hành phục hồi sang Region mới |
| Tuần 13   | Tổng kết & Hoàn thiện Báo cáo | - Hoàn thiện REPORT.MD, slide, và video demo cuối kỳ [cite: 134, 142] |

---

## 2. Kế hoạch Chi tiết cho từng Nhiệm vụ

### 2.1. Nhiệm vụ 1: Phân tích và Loại bỏ Điểm lỗi (High Availability)

**Mục tiêu:** Đạt được High Availability (HA) bằng cách loại bỏ các **Single Points of Failure (SPOF)**  

**Các bước thực hiện (Tuần 9-10):**  
1. **Vẽ sơ đồ:** Phân tích sơ đồ "Bộ Xương" và xác định các SPOF (ví dụ: 1 instance service, 1 CSDL, 1 Availability Zone).  
2. **Đề xuất giải pháp (Kiến trúc Multi-AZ):**  
   - Sử dụng **Application Load Balancer (ALB)** để phân tải và tự động chuyển hướng traffic khi có lỗi.  
3. **Cấu hình các service:**  
   - ECS/EC2 chạy với ít nhất **2 instances** và triển khai trên **2 Availability Zone (Multi-AZ)** khác nhau.  
4. **Cấu hình CSDL:**  
   - RDS ở chế độ **Multi-AZ**.  
5. **Viết ADR (Architectural Decision Record):**  
   - Ghi lại quyết định chọn **CSDL Multi-AZ**.  
   - Phân tích trade-off (Cost vs. Reliability): ví dụ, "chi phí tăng gấp đôi... nhưng RTO gần như bằng 0", "chấp nhận đánh đổi về chi phí và một chút hiệu năng".

---

### 2.2. Nhiệm vụ 2: Thực hành Chaos Engineering

**Mục tiêu:** Kiểm chứng khả năng **tự phục hồi** của hệ thống và các pattern tăng độ tin cậy (Retry, Circuit Breaker, Timeout)  

**Các bước thực hiện (Tuần 11):**  
1. **Chọn công cụ:** Sử dụng **AWS Fault Injection Simulator (FIS)**.  

2. **Thiết kế Kịch bản 1: "Tắt Service Instance"**  
   - Giả lập lỗi: Dùng FIS để **terminate** 1 trong 2 instance TripService đang chạy.  
   - Mục tiêu kiểm chứng: **ALB tự động chuyển hướng traffic** và **ECS tự động khởi động lại instance mới**.  

3. **Thiết kế Kịch bản 2: "Tăng độ trễ CSDL"**  
   - Giả lập lỗi: Dùng FIS để tiêm lỗi latency 3 giây vào **UserService (PostgreSQL)**.  
   - Mục tiêu kiểm chứng: TripService **kích hoạt pattern Timeout** (ví dụ: 1 giây) và trả lỗi "fail fast" cho người dùng thay vì bị treo.  

4. **Thu thập kết quả:** Quay video/lưu log quá trình hệ thống tự phục hồi.  

---

### 2.3. Nhiệm vụ 3: Thiết kế và Thực hành Kịch bản Phục hồi sau Thảm họa (DR)

**Mục tiêu:** Kiểm chứng khả năng phục hồi toàn bộ hệ thống sang **Region khác** khi có "thảm họa"  

**Các bước thực hiện (Tuần 12):**  
1. **Thiết kế Kế hoạch DR (Tài liệu):**  
   - Tính toán **RPO** (Recovery Point Objective): chấp nhận mất bao nhiêu dữ liệu  
   - Tính toán **RTO** (Recovery Time Objective): chấp nhận downtime trong bao lâu  

2. **Viết Quy trình Phục hồi:** Mô tả chi tiết các bước DR  

3. **Chuẩn bị Kỹ thuật:**  
   - Cấu hình RDS tự động **sao lưu snapshot liên vùng (Cross-Region snapshots)**  
   - Cấu trúc lại code Terraform (IaC) để tái sử dụng, chỉ cần thay đổi biến region  

4. **Thực hành (Mô phỏng):**  
   - Giả lập Region chính (ap-southeast-1) bị sập  
   - **Bước 1 (Restore Data):** Phục hồi CSDL từ snapshot mới nhất ở Region dự phòng (ap-northeast-1)  
   - **Bước 2 (Restore Infra):** Chạy script Terraform để tạo lại toàn bộ hạ tầng (VPC, ALB, ECS...) tại Region dự phòng  
   - **Bước 3 (Redirect Traffic):** Cập nhật DNS để trỏ về ALB ở Region mới  

5. **Thu thập kết quả:**  
   - Đo thời gian thực tế để hoàn thành (**RTO thực tế**)  
   - Quay video quá trình phục hồi  

---
