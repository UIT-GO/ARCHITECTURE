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

**Mục tiêu:**  
Xây dựng kịch bản (**k6**), tìm điểm nghẽn (bottleneck) của SUT (System Under Test), và đo lường giới hạn (RPS) của hệ thống.

---

#### ⚠️ Lưu ý về môi trường
- Máy Load Generator local: **8 core / 16GB RAM** → chỉ phù hợp cho Smoke Test hoặc kiểm tra chức năng dưới tải thấp.  
- Để đạt mục tiêu **hyper-scale (1000+ VUs)**, nhóm triển khai Load Testing theo 2 giai đoạn:

---

#### Giai đoạn A: Baseline & Tối ưu trên Môi trường Giới hạn (Tuần 10)

**Mục đích:** Xác định baseline và điểm nghẽn ban đầu của kiến trúc "Bộ Xương" (chưa tối ưu).

##### Kịch bản 1 — Đặt xe (Functional Test)
- Mô phỏng **50 → 100 Virtual Users** tăng dần.  
- **Đo lường:**  
  - P95 Latency  
  - Tỷ lệ lỗi (Error Rate) → kiểm tra tính ổn định.

##### Kịch bản 2 — Cập nhật vị trí (Smoke Test)
- Mô phỏng **200 → 300 VUs** liên tục gửi vị trí mới.  
- **Đo lường:**  
  - CPU Load Generator → xác định giới hạn máy test.  
  - CPU CSDL / Redis → tìm điểm nghẽn đầu tiên.

**Sản phẩm:** Báo cáo **Baseline**, làm căn cứ cho Tối ưu hóa (Nhiệm vụ 3).

---

#### Giai đoạn B: Target Scale Testing (Tuần 13)

**Mục đích:** Kiểm chứng khả năng chịu tải **hyper-scale** của hệ thống đã tối ưu (Caching, Auto Scaling,…).

**Yêu cầu:**  
- Bắt buộc sử dụng **Load Testing phân tán** (Distributed), ví dụ:  
  - JMeter Master-Slave trên 4–5 AWS t2.medium instances  
  - Hoặc dịch vụ Cloud Load Testing
##### Kịch bản 1 — Đặt xe (Target Scale)
- Mô phỏng **1,000 Virtual Users** hoặc đủ tải đạt **500 req/s**.  
- **Đo lường:**  
  - P99 Latency  
  - Error Rate  
  - Queue Depth của SQS

##### Kịch bản 2 — Cập nhật vị trí (Write-heavy Scale)
- Mô phỏng **10,000 VUs** hoặc đủ tải đạt **>1,000 req/s**.  
- **Đo lường:**  
  - Error Rate  
  - CPU Redis/DynamoDB  
  - Throughput tối đa (req/s)

**Sản phẩm:**  
- Biểu đồ **Before vs After Optimization**: Latency, Error Rate, Throughput, CPU/Memory  
- Chứng minh giá trị của kiến trúc đã thiết kế.

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
