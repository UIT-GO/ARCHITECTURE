# ADR 005: Lựa Chọn Cơ Chế Giao Tiếp Real-Time (WebSocket)
**Trạng thái:** Đã chấp nhận

---

## 📌 Bối cảnh
Ứng dụng UIT-Go là một hệ thống gọi xe, yêu cầu **cập nhật vị trí của tài xế trên bản đồ theo thời gian thực** cho hành khách.  

**User Story 3:** *"Trong lúc chờ xe, tôi muốn thấy được vị trí của tài xế đang di chuyển trên bản đồ theo thời gian thực để biết khi nào họ sẽ tới."*

Yêu cầu này cần:
- Kết nối **song công (bi-directional)**.
- Kết nối **duy trì (persistent)**.
- Độ trễ thấp.

---

## 🧩 Quyết định
- Sử dụng **WebSocket** để xử lý giao tiếp **Real-time** giữa client (ứng dụng di động/web) và Backend (DriverService, TripService).  
- **DriverService:** nhận dữ liệu vị trí liên tục từ tài xế qua API (có thể là RESTful).  
- **TripService:** sử dụng WebSocket để đẩy vị trí cập nhật từ DriverService đến hành khách **theo thời gian thực**.

---

## 🔍 Các lựa chọn đã cân nhắc
1. **Polling:** Client liên tục gửi request HTTP để kiểm tra cập nhật.  
   - Nhược điểm: tốn tài nguyên, độ trễ cao, không hiệu quả.  
2. **Long Polling:** Giữ kết nối mở.  
   - Nhược điểm: vẫn có độ trễ và phức tạp khi mở rộng (scaling).

---

## ⚖️ Đánh đổi (Trade-offs)

### ✅ Lợi ích
- Cung cấp **kết nối duy trì, song công** với độ trễ cực thấp.  
- Đáp ứng yêu cầu nghiệp vụ cốt lõi về **Real-time Tracking**.

### ⚠️ Chi phí
- Tăng độ phức tạp ở tầng **Gateway và Load Balancer** (cần hỗ trợ sticky session hoặc WebSocket protocol).  
- Chi phí duy trì **kết nối mở** (persistent connections) cao hơn so với HTTP thông thường.

---

## 📝 Kết luận
Việc sử dụng **WebSocket** giúp UIT-Go đáp ứng hiệu quả yêu cầu **cập nhật vị trí tài xế theo thời gian thực**, tăng trải nghiệm người dùng, đồng thời phải chấp nhận chi phí vận hành và thiết kế cao hơn.
