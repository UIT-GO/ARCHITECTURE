# 📄 Báo cáo Kế hoạch Chi tiết: Module B (Reliability & High Availability)

**Thành viên:**  
- Võ Minh Kiệt (22520727)  
- Võ Mai Nguyên (22520991)  

**Module đã chọn:** Module B: Thiết kế cho Reliability & High Availability  
**Vai trò đảm nhận:** Kỹ sư Đảm bảo Độ tin cậy (Site Reliability Engineer - SRE)  

---

## 3. Mục tiêu tổng quan và Bối cảnh 🚀

- Thiết kế hệ thống có khả năng chống chịu và tự phục hồi để bảo vệ UIT-Go khỏi **mất mát $5,000/giờ$ do downtime** trong giai đoạn "Scale or Die".  
- **RTO Mục tiêu:** ≤ 30 phút  
- **RPO Mục tiêu:** ≤ 5 phút  
- **Chiến lược DR:** Pilot Light (tối ưu giữa chi phí và tốc độ phục hồi)

---

## 1. Lộ trình Thực hiện (Timeline Giai đoạn 2)

| Tuần       | Nhiệm vụ chính                                    | Sản phẩm đầu ra (Deliverables) |
|------------|--------------------------------------------------|--------------------------------|
| Tuần 9-10  | Nhiệm vụ 1: Phân tích & Loại bỏ Điểm lỗi (HA trên EKS) | - Sơ đồ kiến trúc Multi-AZ EKS (cập nhật ARCHITECTURE.MD) <br> - Cấu hình Terraform EKS/RDS Multi-AZ <br> - File ADR-001 phân tích Trade-off (Cost vs. RTO/Multi-AZ) |
| Tuần 11    | Nhiệm vụ 2: Thực hành Chaos Engineering         | - Kịch bản test trên AWS FIS và Kubernetes (xóa Pod/terminate Node) <br> - Log/video chứng minh hệ thống tự phục hồi (EKS ReplicaSet) và pattern Timeout |
| Tuần 12    | Nhiệm vụ 3: Thiết kế & Thực hành DR (Pilot Light) | - Tài liệu Kế hoạch DR (tính toán RTO/RPO) <br> - Terraform script cho Region dự phòng (Pilot Light Infra) <br> - Log/video thực hành phục hồi sang Region mới (đo RTO thực tế) |
| Tuần 13    | Tổng kết & Hoàn thiện Báo cáo                   | - Hoàn thiện REPORT.MD (Phần cốt lõi là Trade-off và ADR) <br> - Slide, video demo cuối kỳ |

---

## 2. Kế hoạch Chi tiết cho từng Nhiệm vụ

### 2.1. Nhiệm vụ 1: Phân tích và Loại bỏ Điểm lỗi (High Availability)

**Mục tiêu:** Triển khai Redundancy và Automated Failover

- **Vẽ sơ đồ:** Xác định các SPOF (ví dụ: CSDL Single-AZ, EKS Node Group Single-AZ)  
- **Đề xuất giải pháp (Kiến trúc Multi-AZ EKS):**  
  - Sử dụng ALB để phân phối traffic  
  - Cấu hình EKS Node Groups để chạy các service (Deployment) trên ít nhất 2 AZ  
  - Cấu hình CSDL: RDS Multi-AZ (Synchronous Replication)  
- **Viết ADR (Trade-off):** Ghi lại quyết định Multi-AZ RDS  
  - Phân tích Trade-off: Chi phí tăng gấp đôi để đạt RTO gần như bằng 0

---

### 2.2. Nhiệm vụ 2: Thực hành Chaos Engineering

**Mục tiêu:** Kiểm chứng khả năng tự phục hồi (Automatic Recovery) và pattern Graceful Degradation  

- **Chọn công cụ:** AWS Fault Injection Simulator (FIS)  

**Kịch bản 1: "Terminate Node/Pod" (Kiểm chứng Self-Healing)**  
- Giả lập lỗi: Dùng FIS để terminate 1 Worker Node (hoặc `kubectl delete pod`)  
- **Mục tiêu kiểm chứng:**  
  - ALB chuyển hướng traffic  
  - EKS (ReplicaSet) tự động khởi động lại Pod ở AZ khỏe mạnh để duy trì công suất  

**Kịch bản 2: "Tăng độ trễ CSDL" (Kiểm chứng Timeout)**  
- Giả lập lỗi: Dùng FIS tiêm lỗi latency 3 giây vào CSDL của UserService  
- **Mục tiêu kiểm chứng:**  
  - TripService kích hoạt pattern Timeout (1 giây) và trả lỗi "fail fast" thay vì bị treo lâu  
  - Core Functions vẫn hoạt động bình thường  

**Thu thập kết quả:** Quay video/lưu log quá trình hệ thống phục hồi (đo RTO thực tế của sự cố)

---

### 2.3. Nhiệm vụ 3: Thiết kế và Thực hành Kịch bản Phục hồi sau Thảm họa (DR)

**Mục tiêu:** Kiểm chứng khả năng phục hồi toàn bộ hệ thống sang Region khác (Multi-Region DR)  

**Thiết kế Kế hoạch DR (Tài liệu)**  
- Tính toán **RPO ≤ 5 phút** và **RTO ≤ 30 phút**  
- Xác nhận chiến lược **Pilot Light** (chỉ giữ RDS Read Replica và EKS Control Plane/1 node)

**Chuẩn bị Kỹ thuật (IaC & Replication)**  
- Cấu hình **RDS Cross-Region Replica** hoặc Snapshot để đáp ứng RPO  
- Sử dụng **Terraform** để định nghĩa hạ tầng Pilot Light ở Region dự phòng

**Thực hành (Mô phỏng Failover)**  
1. **Restore Data:** Phục hồi CSDL (Promote Read Replica)  
2. **Restore Infra/Scale Up:** Chạy Terraform script để Scale Up EKS Cluster (từ 1 node lên 3+ node) và tạo ALB mới  
3. **Redirect Traffic:** Cập nhật Route 53 DNS Failover để trỏ về ALB ở Region mới

**Thu thập kết quả:**  
- Đo thời gian thực tế để hoàn thành (**RTO thực tế**)  
- Quay video quá trình phục hồi
