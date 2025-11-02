# Module A: Thiết kế Kiến trúc cho Scalability & Performance

## Use Case 1: Đặt xe (Booking)

### 1. Phân tích và Bảo vệ Lựa chọn Kiến trúc

**Vấn đề (Problem):**  
- Nếu dùng HTTP đồng bộ (synchronous), luồng TripService → DriverService sẽ bị **blocking**.  
- TripService phải chờ DriverService tìm tài xế, và DriverService phải chờ tài xế xác nhận (vài giây đến hàng chục giây).  
- Khi tải tăng đột biến, TripService sẽ cạn kiệt connection, gây tắc nghẽn toàn hệ thống và sập.

**Giải pháp 1: Tách rời (Decouple) bằng Giao tiếp Bất đồng bộ (Asynchronous)**  
- **Lựa chọn:** Sử dụng Message Queue (SQS, Kafka, RabbitMQ) giữa TripService và DriverService.  
- **Luồng xử lý:**  
  1. User gọi API đặt xe → TripService.  
  2. TripService tạo record với trạng thái `PENDING` và đẩy event `CreateTripEvent` vào Message Queue.  
  3. TripService trả về HTTP 202 Accepted (hoặc 200 OK) cho User ngay lập tức (< 100ms) với thông báo "Đang tìm tài xế...".  
- **Trade-off:**  
  - Đánh đổi tính nhất quán tức thời (immediate consistency) để lấy **availability** và **scalability**.  
  - User sẽ nhận thông báo sau khi tài xế xác nhận (eventual consistency).  

**Giải pháp 2: Giao tiếp Real-time với WebSocket**  
- **Lựa chọn:** Sử dụng WebSocket full-duplex cho Driver và Customer.  
- **Bảo vệ quyết định:**  
  - **Driver:** DriverService nhận event từ Message Queue → đẩy cuốc mới đến tài xế qua WebSocket → tối ưu thời gian tìm kiếm nhanh nhất.  
  - **Customer:** Khi tài xế xác nhận → TripService đẩy trạng thái `ASSIGNED` qua WebSocket cho Customer.

---

### 2. Kiểm chứng Thiết kế bằng Load Testing

- **Kịch bản:**  
  - Sử dụng k6/JMeter mô phỏng 1.000–5.000 user đồng thời gọi API POST `/trips` trong 1 phút.  
- **Metrics cần theo dõi:**  
  1. **TripService API:** P99 Latency < 200ms, Error Rate = 0%.  
  2. **Message Queue Depth:** Khi tải tăng, queue depth có thể tăng vọt nhưng phải giảm dần. Nếu queue không giảm, DriverService xử lý không kịp → cần tuning.

---

### 3. Hiện thực hóa Kỹ thuật Tối ưu hóa (Tuning)

- **Vấn đề:** DriverService xử lý message không kịp → tồn đọng.  
- **Giải pháp (Tuning): Auto Scaling Group cho DriverService**  
  - Logic:  
    - Nếu số lượng message trong queue > 1000 → thêm instance DriverService.  
    - Nếu số message < 100 → giảm instance để tiết kiệm chi phí.  

---

## Use Case 2: Cập nhật vị trí của Driver

### 1. Phân tích và Bảo vệ Lựa chọn Kiến trúc

**Vấn đề 1: Storage & Search**  
- Cần truy vấn hàng nghìn–hàng chục nghìn tài xế theo bán kính 5km, tốc độ realtime.  
- **Giải pháp:** Redis GEO  
- **Lý do chọn Redis GEO:**  
  1. **Tốc độ:** Lưu trên RAM, lệnh GEOADD/GEORADIUS sub-millisecond.  
  2. **Hiệu quả:** Cập nhật vị trí dễ dàng, API sẵn có.  
  3. **Đơn giản & ổn định:** Không cần triển khai cấu trúc dữ liệu phức tạp, dễ tích hợp microservices.  
  4. **Khả năng mở rộng:** Hỗ trợ cluster, scale horizontal.  

**Vấn đề 2: Giao thức cập nhật vị trí**  
- **Giải pháp:** gRPC (Client-side Streaming)  
- **Lý do:**  
  1. **Size tối ưu:** Protobuf nhị phân, vài bytes cho mỗi vị trí.  
  2. **Efficiency:** Stream liên tục trên HTTP/2, tiết kiệm pin và băng thông so với REST.  

---

### 2. Kiểm chứng Thiết kế bằng Load Testing

- **Write Test:** 50.000 tài xế (VUs) mở gRPC stream, gửi vị trí mới mỗi 5s (~10.000 RPS).  
- **Read Test:** 1.000 request đặt xe/s, mỗi request query GEORADIUS (5km) trên Redis.  
- **Metrics:**  
  - Redis CPU Utilization (<100%).  
  - P99 Latency GEORADIUS < 10ms.

---

### 3. Hiện thực hóa Kỹ thuật Tối ưu hóa (Tuning)

1. **Auto Scaling Group cho DriverService:** scale dựa trên CPU hoặc Network Traffic.  
2. **Read Replicas (Bản sao chỉ đọc):**  
   - Primary Node xử lý GEOADD (ghi).  
   - Replica Nodes xử lý GEORADIUS (đọc).  
   - Tách biệt hoàn toàn tải "ghi" và "đọc".  
3. **Sharding (Phân mảnh) Redis Cluster:**  
   - Nếu Primary Node quá tải (CPU 100%), phân mảnh theo Geohash → scale ngang vô hạn.

---

