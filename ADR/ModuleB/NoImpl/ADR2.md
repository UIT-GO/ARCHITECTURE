# ADR 006: Xử lý Tranh chấp Nhận chuyến bằng Redis Atomic Reservation

## Trạng thái (Status)
**Accepted (Design Phase)**
*Lưu ý: Đây là quyết định kiến trúc được phê duyệt cho giai đoạn phát triển tiếp theo (Phase 2) nhằm tối ưu hiệu năng khi số lượng tài xế tăng cao.*

## 1. Bối cảnh (Context)
Trong mô hình gọi xe UIT-Go, một yêu cầu đặt xe (Trip) được phát sóng (broadcast) cho nhiều tài xế trong khu vực lân cận.
Vấn đề **Race Condition (Tranh chấp)** xảy ra khi:
1.  Tài xế A bấm "Nhận chuyến" tại thời điểm `T`.
2.  Tài xế B bấm "Nhận chuyến" tại thời điểm `T + 1ms`.

Nếu xử lý trực tiếp vào Database (RDBMS), cả hai request có thể đọc thấy trạng thái chuyến đi là `FINDING` và cùng cập nhật thành `ACCEPTED`.
* **Hậu quả:** Một chuyến xe có 2 tài xế (Double Booking). Dữ liệu sai lệch, gây trải nghiệm tồi tệ.

## 2. Quyết định (Decision)
Chúng tôi quyết định sử dụng cơ chế **Redis Atomic Reservation** (Đặt chỗ nguyên tử) để làm "cổng gác" (Gatekeeper) trước khi cập nhật vào Database.

* **Cơ chế:** Sử dụng lệnh `SETNX` (Set if Not Exists) của Redis để tạo một khóa (Lock) tạm thời cho `trip_id`.
* **Quy tắc:** Chỉ request nào đặt được key vào Redis thành công (trả về 1) mới được quyền đi tiếp vào Database để gán tài xế. Các request đến sau (trả về 0) sẽ bị từ chối ngay lập tức.

## 3. Phân tích Lựa chọn (Alternatives Analysis)

### Option A: Pessimistic Locking (Database Level)
* **Mô tả:** Sử dụng `SELECT ... FOR UPDATE` để khóa dòng dữ liệu trong bảng `trips`.
* **Đánh giá:** An toàn tuyệt đối nhưng **Hiệu năng rất thấp**. Khi 100 tài xế cùng bấm, 1 người được xử lý, 99 người còn lại sẽ bị treo kết nối Database chờ đợi (Blocking). Dễ gây sập Database.
* **Kết luận:** LOẠI BỎ.

### Option B: Optimistic Locking (Database Level - Versioning)
* **Mô tả:** Dùng cột `version` để kiểm tra. `UPDATE ... WHERE version = current_ver`.
* **Đánh giá:** Tốt hơn Option A, không gây blocking. Tuy nhiên, nếu lượng tranh chấp quá lớn (High Contention), Database vẫn phải chịu tải của hàng trăm câu lệnh `UPDATE` thất bại.
* **Kết luận:** PHÙ HỢP VỚI QUY MÔ NHỎ/TRUNG BÌNH.

### Option C: Redis Atomic Reservation [ĐƯỢC CHỌN]
* **Mô tả:** Dùng Redis làm lớp khóa phân tán (Distributed Lock) nhẹ.
* **Đánh giá:**
    * **Ưu điểm:** Tốc độ xử lý cực nhanh (in-memory). Redis là Single-threaded nên các lệnh là tuần tự tuyệt đối (Atomic). Giảm tải tối đa cho Database (Database chỉ nhận duy nhất 1 lệnh UPDATE thành công).
    * **Nhược điểm:** Thêm phụ thuộc vào hạ tầng Redis.
* **Kết luận:** **CHẤP NHẬN** để đáp ứng yêu cầu High Availability và High Throughput của Module B.

## 4. Thiết kế Chi tiết (Detailed Design)

Quy trình xử lý một request "Nhận chuyến" sẽ như sau:

1.  **Bước 1 (Redis Reservation):**
    * Server gọi lệnh Redis: `SET trip_lock:{trip_id} {driver_id} NX EX 10`
    * `NX`: Chỉ set nếu key chưa tồn tại (Not Exists).
    * `EX 10`: Key tự hết hạn sau 10 giây (để tránh Deadlock nếu Server bị crash giữa chừng).

2.  **Bước 2 (Kiểm tra kết quả):**
    * **Nếu Redis trả về `OK` (True):** Tài xế này đã thắng cuộc đua. -> Đi tiếp sang Bước 3.
    * **Nếu Redis trả về `Nil` (False):** Đã có người khác nhận trước. -> Trả lỗi "Chuyến đi đã có người nhận" ngay lập tức.

3.  **Bước 3 (Database Persistence):**
    * Cập nhật PostgreSQL: `UPDATE trips SET driver_id = ..., status = 'ACCEPTED' WHERE id = ...`
    * Vì đã qua bước 1, bước này gần như chắc chắn thành công 100%.

4.  **Bước 4 (Cleanup - Optional):**
    * Xóa key Redis hoặc để nó tự hết hạn (TTL).

## 5. Hệ quả & Đánh đổi (Consequences & Trade-offs)

| Tiêu chí | Phân tích tác động |
| :--- | :--- |
| **Performance** | **Tích cực:** Phản hồi cực nhanh (Sub-millisecond latency). Database không bao giờ bị quá tải bởi các request tranh chấp. |
| **Reliability** | **Tích cực:** Loại bỏ hoàn toàn Race Condition. Cơ chế TTL (Time-to-live) giúp hệ thống tự phục hồi nếu có lỗi. |
| **Complexity** | **Tiêu cực:** Phải quản lý thêm kết nối Redis và xử lý các trường hợp biên (edge cases) như Redis bị sập (lúc đó có thể fallback về Optimistic Locking). |
| **Infrastructure** | **Yêu cầu:** Cần Redis Cluster cấu hình High Availability để tránh Redis trở thành điểm lỗi đơn lẻ (SPOF). |

## 6. Kế hoạch Hiện thực hóa (Implementation Roadmap)

* **Phase 2:**
    * Implement logic Redis `SETNX` trong `DriverService`.
    * Cấu hình Redis Cluster trên AWS ElastiCache.
    * Viết Lua Script nếu cần thực hiện nhiều lệnh Redis cùng lúc để đảm bảo tính nguyên tử cao hơn.
