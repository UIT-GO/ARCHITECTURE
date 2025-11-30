# ADR 005: Sử dụng Saga Orchestration cho Giao dịch Phân tán (Distributed Transactions)

## Trạng thái (Status)
**Accepted (Design Phase)**
*Lưu ý: Đây là quyết định kiến trúc nền tảng được phê duyệt cho giai đoạn phát triển tiếp theo (Phase 2), nhằm giải quyết vấn đề nhất quán dữ liệu (Data Consistency) khi hệ thống mở rộng.*

## 1. Bối cảnh (Context)
Trong kiến trúc Microservices hiện tại của UIT-Go, chúng ta tuân thủ nguyên tắc **Database per Service**. Mỗi service (`TripService`, `DriverService`) quản lý một cơ sở dữ liệu riêng biệt để đảm bảo tính độc lập và khả năng mở rộng.

Tuy nhiên, quy trình nghiệp vụ **"Tài xế nhận chuyến" (Driver Acceptance)** yêu cầu cập nhật dữ liệu trên hai service khác nhau một cách đồng bộ:
1.  **DriverService:** Cập nhật trạng thái tài xế từ `ONLINE` sang `BUSY` (khóa tài xế).
2.  **TripService:** Cập nhật trạng thái chuyến đi từ `FINDING` sang `ACCEPTED` và gán `driver_id`.

**Vấn đề:**
Chúng ta không thể sử dụng ACID Transaction truyền thống (Local Transaction) để đảm bảo tính nguyên tử (Atomicity) cho cả hai hành động này vì chúng nằm trên 2 DB vật lý khác nhau.
* **Rủi ro:** Nếu `DriverService` cập nhật thành công (Tài xế Bận) nhưng `TripService` bị lỗi (do Database quá tải hoặc lỗi logic), tài xế sẽ bị kẹt ở trạng thái `BUSY` mà thực tế không có chuyến đi nào được gán.
* **Hậu quả:** Dữ liệu hệ thống bị sai lệch (**Ghost Booking**), gây thiệt hại tài chính và trải nghiệm tồi tệ cho tài xế.

## 2. Quyết định (Decision)
Chúng tôi quyết định sử dụng **Saga Pattern** theo mô hình **Orchestration (Điều phối)** để quản lý giao dịch phân tán này.

* **Pattern:** Saga Orchestration.
* **Orchestrator (Bộ điều phối):** Service `TripService` sẽ đóng vai trò là "Nhạc trưởng" (Central Coordinator).
* **Cơ chế:** Thực hiện chuỗi các transaction cục bộ (Local Transactions). Nếu một bước bất kỳ thất bại, Orchestrator sẽ chịu trách nhiệm kích hoạt **Compensating Transaction** (Giao dịch bù trừ) để hoàn tác (undo) các bước đã thành công trước đó.

## 3. Phân tích Lựa chọn (Alternatives Analysis)

Chúng tôi đã cân nhắc ba phương án để xử lý vấn đề này:

### Option A: Two-Phase Commit (2PC / XA Transactions)
* **Mô tả:** Sử dụng giao thức khóa phân tán để đảm bảo cả 2 DB cùng commit hoặc cùng rollback.
* **Đánh giá:** Hiệu năng rất thấp do blocking (khóa chờ tài nguyên). Tạo ra điểm nghẽn (Single Point of Failure). Không phù hợp với Database NoSQL (như MongoDB).
* **Kết luận:** LOẠI BỎ (Rejected).

### Option B: Saga Choreography (Dựa trên Sự kiện - Event-based)
* **Mô tả:** Các service giao tiếp thông qua Message Broker (Kafka). Service A làm xong sẽ bắn event, Service B lắng nghe và làm tiếp.
* **Đánh giá:** Giảm sự phụ thuộc (Loose coupling) nhưng **khó theo dõi trạng thái transaction** (Observability kém). Logic nghiệp vụ bị phân tán rải rác, rất khó debug và kiểm soát khi hệ thống lớn (Cyclic Dependency risk).
* **Kết luận:** LOẠI BỎ (Rejected).

### Option C: Saga Orchestration (Dựa trên Điều phối) [ĐƯỢC CHỌN]
* **Mô tả:** Service `TripService` biết chính xác quy trình gồm những bước nào, chịu trách nhiệm gọi các service khác và quyết định khi nào cần Rollback.
* **Đánh giá:**
    * **Ưu điểm:** Dễ dàng quản lý và giám sát, logic tập trung tại một nơi (`TripService`), giảm độ phức tạp khi xử lý các kịch bản lỗi (Error Handling).
    * **Nhược điểm:** `TripService` gánh thêm trách nhiệm điều phối.
* **Kết luận:** **CHẤP NHẬN** phương án này vì ưu tiên khả năng kiểm soát độ tin cậy dữ liệu (Data Reliability) - mục tiêu cốt lõi của Module B.

## 4. Thiết kế Luồng xử lý (Workflow Design Blueprint)



Quy trình Saga cho luồng "Tài xế nhận chuyến" được thiết kế như sau:

### 4.1. Luồng Thành công (Happy Path)
1.  **Start:** Tài xế bấm "Nhận chuyến".
2.  **Step 1 (DriverService):** `TripService` gửi lệnh khóa tài xế (`LockDriver`).
    * *Kết quả:* DriverService chuyển trạng thái sang `BUSY`.
3.  **Step 2 (TripService):** `TripService` thực hiện gán chuyến (`AssignDriver`) vào DB nội bộ.
    * *Kết quả:* Chuyến đi chuyển sang `ACCEPTED`.
4.  **End:** Trả về thành công cho tài xế.

### 4.2. Luồng Thất bại & Bù trừ (Failure & Compensation Path)
1.  **Step 1:** `TripService` gửi lệnh `LockDriver` -> **Thành công** (Tài xế đã Bận).
2.  **Step 2:** `TripService` thực hiện `AssignDriver` -> **THẤT BẠI** (Ví dụ: Trip đã bị user hủy trước đó 1ms).
3.  **Compensation Trigger:** `TripService` bắt được lỗi (Catch Exception).
4.  **Compensation Action (Bù trừ):** `TripService` gửi lệnh **`UnlockDriver(driverId)`** sang `DriverService`.
    * *Kết quả:* `DriverService` chuyển trạng thái tài xế về lại `ONLINE`.
5.  **End:** Trả về thông báo lỗi cho tài xế, nhưng dữ liệu hệ thống vẫn nhất quán (Tài xế Online, Trip Cancelled).

## 5. Hệ quả & Đánh đổi (Consequences & Trade-offs)

| Tiêu chí | Phân tích tác động |
| :--- | :--- |
| **Data Consistency** | **Tích cực:** Đảm bảo tính nhất quán cuối cùng (Eventual Consistency). Loại bỏ hoàn toàn lỗi dữ liệu sai lệch giữa các services. |
| **Development Complexity** | **Tiêu cực:** Chi phí phát triển dự kiến tăng khoảng 30-40% do phải viết code cho các hàm Rollback/Compensation và quản lý State Machine. |
| **Performance** | **Trung tính:** Chậm hơn Choreography một chút do giao tiếp tập trung, nhưng nhanh hơn 2PC rất nhiều vì không lock resource dài hạn. |
| **Testability** | **Tích cực:** Dễ dàng viết Unit Test để mô phỏng các kịch bản lỗi vì logic luồng đi nằm gọn trong Orchestrator. |

## 6. Kế hoạch Hiện thực hóa (Implementation Roadmap)

Trong giai đoạn tiếp theo (Phase 2), sau khi hoàn thiện hạ tầng HA (Module B Phase 1), nhóm sẽ tiến hành implement thiết kế này:

* **Công nghệ:** Java Spring Boot.
* **Kỹ thuật:** Sử dụng **State Design Pattern** để quản lý trạng thái giao dịch (`STARTED`, `DRIVER_LOCKED`, `FAILED`, `ROLLED_BACK`).
* **Giao tiếp:** Sử dụng gRPC cho các lệnh nội bộ để đảm bảo tốc độ thấp nhất (Low latency).
