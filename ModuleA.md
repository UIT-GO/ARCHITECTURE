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
# ĐẶT XE
![ĐẶT XE](Image/Đặtxe.png)
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


# ⚙️ Nguyên tắc Gửi Dữ liệu vị trí của Driver

![Cập nhật vị trí](Image/logiccapnhatvitri.png)

Ứng dụng **chỉ gửi vị trí mới** lên `DriverService` khi **một trong hai điều kiện sau** được thỏa mãn:

1. **Đã trôi qua hơn 3–5 giây** kể từ lần gửi cuối
    Hình này là nguyên tắc chu kỳ thời gian gửi
   ![Cập nhật vị trí](Image/logiccapnhatvitri.png)
3. **Hoặc** tài xế đã di chuyển **hơn 10–20 mét** so với vị trí trước đó  

> 👉 Nhờ vậy, khi tài xế đứng yên (kẹt xe, dừng đèn đỏ...), ứng dụng **không gửi liên tục** dữ liệu trùng lặp.

---

### 💓 Cơ chế “Heartbeat” Dự phòng

Nếu tài xế **đứng im quá lâu** (trên 2–3 phút), ứng dụng sẽ gửi một **gói “heartbeat”** để báo cho server biết:  
> “Tôi vẫn đang online, chỉ là chưa di chuyển.”

---

### 🚀 Lợi ích

| Lợi ích | Mô tả |
|----------|--------|
| 🔋 Tiết kiệm pin | Không gửi request liên tục khi không cần thiết |
| 🌐 Giảm tải server | Giảm số lượng API call và update vào Redis/MongoDB |
| ⚡ Phản hồi nhanh | Gửi ngay khi tài xế di chuyển đủ xa |
| 🧠 Dễ mở rộng | Có thể tinh chỉnh ngưỡng `distance_threshold` và `time_interval` động theo trạng thái |


---

## ⚡ Tại sao chọn WebSocket cho TripService

WebSocket được chọn để hỗ trợ giao tiếp **hai chiều (bi-directional)** giữa server và client theo **thời gian thực**.

### 🚖 1. Đặc thù của TripService
TripService là trung tâm điều phối giữa:
- 🧍‍♂️ **Người dùng (User)**: tạo và theo dõi chuyến đi  
- 🚗 **Tài xế (Driver)**: nhận cuốc, cập nhật trạng thái và vị trí  

Hệ thống cần cập nhật **liên tục**:
- Khi tài xế **nhận cuốc**, người dùng thấy ngay  
- Khi người dùng **hủy**, tài xế biết ngay  
- Khi tài xế **di chuyển**, vị trí được cập nhật real-time  

---

# 1. 🛡️ UserService (Auth) → PostgreSQL

## 1. Phân tích và Bảo vệ Lựa chọn Kiến trúc

### Phân tích (Nhiệm vụ)
- UserService quản lý **dữ liệu nhạy cảm** và có cấu trúc chặt chẽ (users, profiles, credentials).
- Yêu cầu quan trọng nhất:
  - **Tính nhất quán (Consistency)**
  - **Toàn vẹn dữ liệu (Integrity)**

### Bảo vệ Quyết định (Chọn PostgreSQL)
- Chúng em chọn **CSDL Quan hệ (PostgreSQL)** vì đảm bảo **ACID** (Atomicity, Consistency, Isolation, Durability).
- Khi người dùng đổi mật khẩu, hệ thống phải thực thi **ngay lập tức và chính xác 100%**, không chấp nhận **Eventual Consistency**.
- Dữ liệu người dùng (users và user_profiles) có quan hệ 1-1 rõ ràng:
  - PostgreSQL hỗ trợ **constraints** và **foreign keys** đảm bảo dữ liệu không bị sai lệch.

### Trade-off (Đánh đổi)
- Hy sinh:
  - **Schema flexibility** (tính linh hoạt của cấu trúc)
  - **Khả năng scale-out (ghi) phức tạp hơn**
- Lợi ích:
  - Đổi lấy **tính nhất quán tuyệt đối** và **toàn vẹn dữ liệu**.

## 2. Kiểm chứng Thiết kế bằng Load Testing

### Kịch bản 1 (Read-heavy)
- Mô phỏng **5.000 người dùng đăng nhập (login) đồng thời**.
- Metric: **P99 Latency** của API login.
- Bottleneck: **CPU của CSDL chính**.

### Kịch bản 2 (Write-heavy)
- Mô phỏng **1.000 người dùng đăng ký (register) đồng thời**.
- Metric: **Error Rate** và số lượng **kết nối (connections)** đến CSDL.

## 3. Hiện thực hóa các Kỹ thuật Tối ưu hóa (Tuning)

### Vấn đề
- Load Test cho thấy **CPU của Primary Node bị quá tải** khi có nhiều yêu cầu đọc đồng thời.

### Giải pháp (Tuning)
- **Mở rộng CSDL bằng Read Replicas**
  - Hiện thực:
    - 1 **Primary Node** (ghi)
    - 2 **Read Replica Nodes** (đọc)
  - Logic:
    - **Ghi** (Đăng ký, Đổi mật khẩu) → Primary Node
    - **Đọc** (Đăng nhập, Lấy hồ sơ) → Read Replicas
  - Kết quả:
    - Giảm tải CPU cho Primary Node
    - Tăng throughput cho luồng **read**
    - Đáp ứng yêu cầu của Module A

---
# 🧾 TripService → MongoDB (CSDL Tài liệu)

## Trách nhiệm của Service
- Xử lý logic tạo chuyến đi.
- Quản lý các trạng thái của chuyến: **PENDING**, **ACCEPTED**, **IN_PROGRESS**, **COMPLETED**, v.v.

## Loại dữ liệu
- Một "cuốc xe" (Trip) là **document** có cấu trúc linh hoạt và liên tục phát triển.
- Ví dụ về trạng thái Trip:
  - **Bắt đầu**: `{ user_id, pickup, destination, status: "PENDING" }`
  - **Cập nhật khi được chấp nhận**: `{ ..., driver_id: "xyz", status: "ACCEPTED" }`
  - **Trong quá trình chạy**: `{ ..., route_history: [...], status: "IN_PROGRESS" }`
  - **Kết thúc**: `{ ..., final_fare: 10, rating: 5, status: "COMPLETED" }`

## Lý do chọn MongoDB

### 1. Schema Linh hoạt (Flexible Schema)
- MongoDB không yêu cầu định nghĩa tất cả các cột từ đầu.
- Dễ dàng thêm trường mới (rating, comment...) mà không cần **ALTER TABLE**.
- Hỗ trợ phát triển nhanh, thích hợp với các tính năng mới liên tục.

### 2. Tối ưu cho Đọc (Read Optimization)
- Toàn bộ thông tin về một cuốc xe có thể lưu trong một document duy nhất.
- Khi cần xem chi tiết, chỉ cần **1 thao tác read** thay vì JOIN nhiều bảng như trong SQL.
- Giúp giảm độ trễ và tăng hiệu năng truy vấn.

### 3. Khả năng Mở rộng (Scalability)
- MongoDB hỗ trợ scale ngang (sharding) dễ dàng.
- Phù hợp khi số lượng cuốc xe tăng lên hàng triệu, hàng tỷ.

## Kết luận
- MongoDB được chọn vì TripService ưu tiên **linh hoạt của cấu trúc** và **tốc độ đọc/ghi** cho các đối tượng (tài liệu) độc lập.




