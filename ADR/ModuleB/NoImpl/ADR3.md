# ADR 003: Cơ chế Idempotency Key cho Trip & Auth Service

## Trạng thái (Status)
**Accepted (Design Phase)**
*Lưu ý: Quyết định này nhằm đảm bảo độ tin cậy (Reliability) cho các thao tác ghi quan trọng khi mạng không ổn định.*

## 1. Bối cảnh (Context)
Hệ thống hiện tại gồm 3 microservices chính: `AuthService`, `TripService`, `DriverService`.
Trong môi trường mạng di động (3G/4G) không ổn định, Client thường xuyên phải thực hiện cơ chế **Retry** (gửi lại request) khi bị Timeout.

**Vấn đề cụ thể:**
1.  **TripService:** Nếu User gửi lại request `POST /trips` nhiều lần, hệ thống có thể tạo ra nhiều chuyến xe trùng lặp (Duplicate Trips) cho cùng một nhu cầu di chuyển.
2.  **AuthService:** Nếu User gửi lại request `POST /register`, hệ thống có thể báo lỗi trùng lặp email thay vì thông báo đăng ký thành công.

## 2. Quyết định (Decision)
Chúng tôi áp dụng pattern **Idempotency Key** sử dụng Redis cho các API thay đổi trạng thái (Non-safe methods: POST, PUT, PATCH).

* **Phạm vi áp dụng:**
    * `TripService`: API Đặt xe (`POST /trips`).
    * `AuthService`: API Đăng ký (`POST /register`).
    * `DriverService`: API Nhận chuyến (`POST /accept`).

* **Cơ chế:** Client gửi header `X-Idempotency-Key` (UUID). Server lưu kết quả xử lý của request đầu tiên vào Redis và trả lại y hệt cho các request tiếp theo có cùng Key.

## 3. Phân tích Kỹ thuật (Technical Analysis)

### Tại sao cần thiết cho TripService?
Đây là nghiệp vụ cốt lõi. Việc tạo ra 2 chuyến xe cùng lúc không chỉ làm sai lệch dữ liệu thống kê mà còn dẫn đến việc điều phối sai tài xế (2 tài xế cùng đến đón 1 khách).
Idempotency đảm bảo tính chất **Exactly-once creation** (Chỉ tạo đúng 1 lần).

### Tại sao cần thiết cho AuthService?
Cải thiện User Experience (UX). Nếu client retry đăng ký, server nên trả về thông tin User đã tạo thành công trước đó (cùng token đăng nhập) thay vì ném ra lỗi `Duplicate Entry`.

## 4. Thiết kế Luồng (Workflow)

1.  **Client** sinh UUID: `123e4567-e89b...`
2.  **TripService** nhận request `POST /trips` với Header `X-Idempotency-Key: 123...`.
3.  **Check Redis:** Key `idemp:123...` có tồn tại?
    * **NO:**
        * Tạo chuyến xe trong DB.
        * Lưu kết quả (JSON Trip) vào Redis với Key `idemp:123...` (TTL 24h).
        * Trả về 201 Created.
    * **YES:**
        * Lấy JSON từ Redis trả về ngay lập tức.
        * Không tạo chuyến xe mới.

## 5. Hệ quả (Consequences)
* **Tích cực:** Hệ thống trở nên tin cậy tuyệt đối trước lỗi mạng (Network Failures). Không bao giờ có chuyện "Duplicate Trip".
* **Tiêu cực:** Tăng chi phí lưu trữ Redis và thêm logic xử lý ở API Gateway/Interceptor.
