# 📄 Báo cáo Kế hoạch Chi tiết: Module A (Scalability & Performance)

**Thành viên:** [Võ Minh Kiệt (22520727)], [Võ Mai Nguyên(22520991)]  
**Module đã chọn:** Module A: Thiết kế Kiến trúc cho Scalability & Performance  
**Vai trò đảm nhận:** Kỹ sư Kiến trúc Hệ thống (System Architect)  

---

## 3. Mục tiêu tổng quan
Trình bày kế hoạch chi tiết để **thiết kế một kiến trúc có khả năng đạt tới "hyper-scale"**, thay vì chỉ tinh chỉnh hệ thống hiện có.  
Kế hoạch này tập trung vào 3 nhiệm vụ chính:  
1. Phân tích và Bảo vệ Lựa chọn Kiến trúc (Phân tích Trade-off)  
2. Kiểm chứng Thiết kế bằng Load Testing (Tìm điểm nghẽn)  
3. Hiện thực hóa các Kỹ thuật Tối ưu hóa (Tuning) dựa trên kết quả test  

---

## 1. Lộ trình Thực hiện (Timeline Giai đoạn 2)

| Tuần       | Nhiệm vụ chính                                | Sản phẩm đầu ra (Deliverables) |
|-----------|-----------------------------------------------|--------------------------------|
| Tuần 9-10 | Nhiệm vụ 1 & 2: Phân tích Kiến trúc & Kiểm thử Baseline | - File ADR phân tích trade-off (ví dụ: SQS vs. Kafka, Redis vs. DynamoDB) [cite: 131, 70] <br> - Kịch bản Load Testing (k6/JMeter) hoàn chỉnh cho các luồng chính <br> - Báo cáo kết quả Load Test (Baseline - "Trước khi Tối ưu") |
| Tuần 11-12 | Nhiệm vụ 3: Hiện thực hóa Tối ưu hóa (Tuning) | - Code (Terraform, Application) hiện thực hóa các kỹ thuật tối ưu [cite: 6] <br> - Ví dụ: ElastiCache [7], Auto Scaling Group [8], Read Replicas [9] |
| Tuần 13   | Nhiệm vụ 2 (Lặp lại) & Tổng kết             | - Chạy lại load test để có kết quả "Sau khi Tối ưu" <br> - Biểu đồ so sánh Trước vs. Sau [10] <br> - Hoàn thiện REPORT.MD [cite: 134], slide, và video demo [cite: 142] |

---

## 2. Kế hoạch Chi tiết cho từng Nhiệm vụ

### 2.1. Nhiệm vụ 1: Phân tích và Bảo vệ Lựa chọn Kiến trúc

**Mục tiêu:** Phân tích các luồng nghiệp vụ quan trọng và bảo vệ các quyết định thiết kế nền tảng bằng các **trade-off**  

**Các bước thực hiện (Tuần 9-10):**  

1. **Phân tích Luồng 1 (Đặt xe):**  
   - **Vấn đề:** Luồng đặt xe có thể bị tăng đột biến (thundering herd). Nếu TripService gọi DriverService đồng bộ (synchronous), hệ thống sẽ sập.  
   - **Giải pháp đề xuất:** Sử dụng mô hình giao tiếp **bất đồng bộ** với **SQS** (hoặc Kafka/RabbitMQ).  

2. **Phân tích Luồng 2 (Cập nhật vị trí):**  
   - **Vấn đề:** Luồng "write-heavy" (ghi nhiều) với hàng chục ngàn request/giây.  
   - **Giải pháp đề xuất:** Phân tích lựa chọn giữa **gRPC (hiệu năng cao)** vs. **REST (đơn giản)** và **Redis GEO (tốc độ)** vs. **DynamoDB+Geohash (scale/cost)**.  

3. **Viết ADR (Architectural Decision Record):**  
   - Ghi lại các quyết định trên vào thư mục `ADR/`.  
   - Mỗi ADR phân tích rõ trade-off: **Cost vs. Performance, Consistency vs. Availability**.  

---

### 2.2. Nhiệm vụ 2: Kiểm chứng Thiết kế bằng Load Testing

**Mục tiêu:** Xây dựng kịch bản (sử dụng k6), tìm điểm nghẽn (bottleneck), và đo lường giới hạn (RPS) của hệ thống  

**Các bước thực hiện (Tuần 10 & 12):**  

1. **Chọn công cụ:** Sử dụng **k6** hoặc **JMeter**  

2. **Thiết kế Kịch bản 1 (Đặt xe - "Ramp-up VUs"):**  
   - Mô phỏng **1000 Virtual Users** tăng dần, đồng thời gọi API đặt xe của TripService.  
   - **Đo lường:** P99 Latency, Tỷ lệ lỗi (Error Rate), Độ sâu hàng đợi (Queue Depth) của SQS.  

3. **Thiết kế Kịch bản 2 (Cập nhật vị trí - "Write-heavy"):**  
   - Mô phỏng **10.000 VUs** liên tục gửi vị trí mới (ví dụ: mỗi 5 giây).  
   - **Đo lường:** P99 Latency, Tỷ lệ lỗi, CPU Utilization của Redis/DynamoDB.  

4. **Thực thi:**  
   - **Lần 1 (Tuần 10):** Chạy test trên "Bộ Xương" (chưa tối ưu) để lấy số liệu **Baseline** và tìm điểm nghẽn.  
   - **Lần 2 (Tuần 12):** Chạy lại chính xác kịch bản trên hệ thống đã được tối ưu.  

---

### 2.3. Nhiệm vụ 3: Hiện thực hóa các Kỹ thuật Tối ưu hóa (Tuning)

**Mục tiêu:** Dựa trên kết quả từ load test (Tuần 10), áp dụng các kỹ thuật "tuning" cụ thể  

**Các bước thực hiện (Tuần 11-12):**  

1. **Nếu Điểm nghẽn là Tải "Đọc CSDL" (UserService bị gọi quá nhiều):**  
   - **Giải pháp:** Áp dụng **Caching (ElastiCache)** cho dữ liệu ít thay đổi  
   - **Hiện thực:** Cache hồ sơ người dùng (user profile) vào **Redis**  

2. **Nếu Điểm nghẽn là Tải "Đột biến" (TripService bị quá tải):**  
   - **Giải pháp:** Thiết lập **Auto Scaling Group** cho service  
   - **Hiện thực:** Cấu hình ECS Auto Scaling dựa trên CPU hoặc độ sâu hàng đợi SQS  

3. **Nếu Điểm nghẽn là Tải "Đọc CSDL" (TripService DB bị chậm khi truy vấn lịch sử):**  
   - **Giải pháp:** Thực hiện chiến lược mở rộng CSDL như **Read Replicas**  
   - **Hiện thực:** Cấu hình **RDS Read Replicas**, cập nhật code TripService để trỏ các truy vấn "read" sang Replicas  

---
