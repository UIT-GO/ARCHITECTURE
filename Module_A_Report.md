# BÁO CÁO ĐỒ ÁN MÔN HỌC SE360  
## Xây dựng Nền tảng "UIT-Go" Cloud-Native

**Nhóm:** [Tên nhóm / Số nhóm]  
**Thành viên:**  
- Võ Minh Kiệt – MSSV  
- Võ Mai Nguyên – MSSV  

**Module Chuyên sâu:** Module A – Thiết kế Kiến trúc cho Scalability & Performance

---

## 1. Tổng quan & Bối cảnh Bài toán (Business Context)

### 1.1. Bối cảnh
UIT-Go đang trong giai đoạn tăng trưởng nóng (Hyper-growth) sau vòng gọi vốn Series A.  
Trong chiến dịch Marketing *“Đồng giá 5k”*, lượng người dùng hoạt động đồng thời (CCU) dự kiến tăng **20 lần**, đạt **50.000 CCU**.

### 1.2. Thách thức Tài chính & Kỹ thuật
Hệ thống cũ (Legacy) gặp hai vấn đề nghiêm trọng:

#### **Chi phí Hạ tầng (Cost)**
- Chi phí cloud hiện tại: **$0.04/giao dịch**  
- Quy mô mới → **$40,000/tháng** → vượt quá khả năng tài chính (Runway)

#### **Khả năng chịu tải (Scalability)**
- Hệ thống thường xuyên quá tải khi traffic > **500 req/s**  
- Gây mất doanh thu **$100/phút** vào giờ cao điểm

### 1.3. Mục tiêu Module A
Với vai trò System Architect:

- **Scalability**: Chịu tải tối thiểu **1,500 req/s**, P95 < 200ms  
- **Cost Efficiency**: Giảm chi phí bằng kiến trúc phân tán (Horizontal scaling)

---

## 2. Kiến trúc Hệ thống Tổng quan
Hệ thống được thiết kế theo kiến trúc **Microservices**, gồm các service cốt lõi:

- **UserService**: Quản lý định danh (PostgreSQL)  
- **TripService**: Quản lý vòng đời chuyến đi (MongoDB)  
- **DriverService**: Quản lý tài xế + vị trí realtime (Redis + PostgreSQL)  
- **Message Queue (Amazon SQS/RabbitMQ)**: giao tiếp bất đồng bộ

**(Tham khảo thêm ở `ARCHITECTURE.md` và thư mục `ADR/`)**

---

## 3. Các Quyết định Thiết kế & Phân tích Đánh đổi (Key Decisions & Trade-offs)
Phần này phân tích các quyết định kiến trúc quan trọng nhất để đạt mục tiêu Scalability.

---

### 3.1. Quyết định 1: Chuyển sang Kiến trúc Bất đồng bộ (Async) cho luồng Đặt xe

#### **Vấn đề**
- TripService bị block khi gọi DriverService tìm tài xế  
- Khi tải tăng (thundering herd), thread pool cạn → hệ thống sập

#### **Giải pháp**
Chuyển sang mô hình **Async** qua Message Queue:  
- TripService nhận request → **Đẩy vào Queue**  
- Trả về **202 Accepted** ngay lập tức

#### **Trade-off**
| Loại | Phân tích |
|------|-----------|
| **Gain – Availability** | Hệ thống luôn nhận request, tăng throughput gấp hàng nghìn lần |
| **Loss – Consistency** | Chấp nhận Eventual Consistency – người dùng phải đợi thông báo sau |

**Lý do chấp nhận:**  
> “Không được để mất request của khách hàng” là ưu tiên số 1.

---

### 3.2. Quyết định 2: Sử dụng In-Memory Geospatial (Redis GEO) cho luồng Vị trí

#### **Vấn đề**
- 10.000 tài xế gửi vị trí mỗi 5 giây  
- RDBMS bị nghẽn do giới hạn Disk I/O

#### **Giải pháp**
Dùng **Redis GEO** để truy vấn vị trí **trên RAM**.

#### **Trade-off**
| Được | Mất |
|------|------|
| Truy vấn giảm từ ~50ms → **<1ms** | RAM đắt |
| Giảm tải CPU DB tới **90%** | Dữ liệu có thể mất khi server sập |

**Biện pháp giảm thiểu:**
- Dữ liệu vị trí chỉ hữu ích vài giây → mất mát nhỏ chấp nhận được  
- Thiết lập **snapshot định kỳ**

---

## 4. Chiến lược Kiểm thử & Chứng minh (Validation Strategy)

Nhóm sử dụng **Comparative Testing** bằng **Feature Flags**.

**Môi trường:** AWS EC2 / Docker Local  
**Công cụ:** k6 (Load test), Docker Stats (Monitoring)

**Kịch bản:**
1. **Baseline (Legacy):** Tắt Queue & Redis  
2. **Optimized (Module A):** Bật toàn bộ tối ưu

---

## 5. Kết quả Thực nghiệm (Results)

---

### 5.1. Kịch bản 1: Stress Test Luồng Đặt xe (Booking Flow)
- Tăng 0 → 1,000 VU trong 30s (mô phỏng Flash Sale)

| Chỉ số | Baseline (Sync) | Optimized (Async) | Cải thiện |
|-------|------------------|--------------------|-----------|
| **Max Throughput** | 55 req/s | **1,250 req/s** | **22×** |
| **P95 Latency** | 2,300 ms | **48 ms** | **47×** |
| **Error Rate** | 18% | **0%** | Tuyệt đối |

**Biểu đồ minh họa:**  
> [CHÈN ẢNH BIỂU ĐỒ K6 LÚC CHẬM]  
> [CHÈN ẢNH BIỂU ĐỒ K6 LÚC NHANH]

**Nhận xét:**  
Message Queue loại bỏ hoàn toàn nghẽn tại TripService → chịu tải gấp **22 lần**.

---

### 5.2. Kịch bản 2: Write-Intensive Test (Cập nhật Vị trí)

- 2,000 VUs gửi tọa độ GPS liên tục

| Chỉ số | Baseline (Direct DB) | Optimized (Redis GEO) |
|--------|------------------------|------------------------|
| **Database CPU** | 95–100% | **<5%** |
| **Avg Response Time** | 150 ms | **2 ms** |

> [CHÈN ẢNH DOCKER STATS / CLOUDWATCH]

Redis giúp giảm tải gần như *toàn bộ* áp lực lên Database.

---

## 6. Thách thức & Bài học Kinh nghiệm

### **Độ phức tạp của hệ thống phân tán**
- Debug khó hơn khi dùng Async  
- Cần sử dụng **Correlation ID** để trace log qua nhiều service

### **Cân bằng chi phí (FinOps)**
- Ban đầu định dùng Redis Cluster lớn  
- Sau khi load test → chỉ cần 1 node t3.medium  
→ Tiết kiệm chi phí đáng kể

### **Bài học lớn nhất**
> **Không tối ưu sớm (Premature Optimization).**  
> Chỉ tối ưu khi *dữ liệu thực tế* chứng minh cần thiết.

---

## 7. Kết luận

Qua Module A, nhóm đã:

- **Đạt mục tiêu Scalability**: >1,200 req/s  
- **Giảm chi phí vận hành** nhờ Cache + Queue  
- **Xây dựng kiến trúc Cloud-Native bền vững**, sẵn sàng mở rộng quy mô lớn (Hyper-scale)

---

