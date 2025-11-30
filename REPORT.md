# UIT-Go System Architecture Document
**Project:** UIT-Go - Cloud-Native Ride Hailing Platform  
**Team:** [Tên Nhóm của bạn]  
**Modules:** A (Scalability), B (Reliability), E (Automation & Cost)  
**Version:** 1.0.0  

---

## 1. Tổng quan Kiến trúc Hệ thống (System Architecture Overview)

Hệ thống UIT-Go được xây dựng theo kiến trúc **Microservices Event-Driven**, tối ưu cho việc xử lý dữ liệu thời gian thực và mở rộng linh hoạt trên AWS.

---

### 1.1. Sơ đồ Logic (Logical Architecture)

![Architecture Diagram](Image/achitecture.jpg)

#### **Mô tả luồng dữ liệu**
**Giao thức giao tiếp:**

- **gRPC Streaming**  
  Dùng cho luồng cập nhật vị trí thời gian thực giữa Driver App / Customer App và Gateway / DriverService, đảm bảo độ trễ thấp nhất.

- **REST API**  
  Dùng cho nghiệp vụ quản lý (User Profile, Booking History) qua HTTP/HTTPS.

- **Message Broker (Kafka)**  
  Đóng vai trò backbone cho giao tiếp bất đồng bộ, xử lý các sự kiện đặt chuyến, cập nhật trạng thái.

- **Service Discovery**  
  Các microservices tự động tìm thấy nhau khi hạ tầng mở rộng hoặc thay đổi.
