# ADR 006: Lựa Chọn Giao Thức Cập Nhật Vị Trí Tài Xế Real-time
**Trạng thái:** Đã chấp nhận

---

## 📌 Bối cảnh
- Luồng nghiệp vụ cốt lõi yêu cầu **tài xế gửi vị trí (latitude/longitude) về DriverService liên tục** (mỗi 1-3 giây).  
- Hệ thống đang trong giai đoạn **"Scale or Die"**, dự kiến tăng trưởng gấp 20 lần (từ 50,000 lên 1,000,000 users).  
- CTO yêu cầu **giảm chi phí hạ tầng từ $0.04 → $0.01 / đơn hàng** và giữ tổng chi phí dưới $10,000/tháng (KPI Cost Efficiency).

---

## 🧩 Quyết định
Sử dụng **gRPC Client-side Streaming** để cập nhật vị trí liên tục từ ứng dụng tài xế (Client) lên **DriverService (Server)**.

---

## 🔍 Phân Tích Các Lựa Chọn

| Lựa chọn | Ưu điểm | Nhược điểm |
|----------|---------|------------|
| **REST/HTTP Polling** | Đơn giản, dễ triển khai | Overhead rất cao (băng thông, CPU), tăng chi phí; không phù hợp Hyper-scale |
| **WebSocket** | Kết nối duy trì, song công (bi-directional), độ trễ thấp | Dữ liệu JSON lớn hơn Protobuf, tốn băng thông và pin hơn gRPC |
| **gRPC Client-side Streaming** | Dữ liệu Protobuf nhị phân siêu nhỏ gọn, tận dụng HTTP/2 (tiết kiệm pin và băng thông) | Client App cần tích hợp thư viện gRPC, phức tạp hơn WebSocket một chút |

**Đánh giá:** gRPC Client-side Streaming là **lựa chọn tối ưu** cho **Scalability** và **Cost Efficiency**.

---

## ⚖️ Bảo Vệ Quyết Định (Trade-offs)

| Lợi ích Đạt được (KPI) | Chi phí/Nhược điểm Chấp nhận |
|------------------------|-----------------------------|
| **Cost Efficiency (KPI cốt lõi):** Giảm chi phí băng thông và tính toán nhờ Protobuf nhị phân (vài bytes mỗi bản ghi vị trí). | **Độ phức tạp phát triển:** Cần tích hợp thư viện gRPC và định nghĩa file `.proto` trên Client App. |
| **Scalability & Performance:** HTTP/2 Streaming với overhead thấp và độ trễ cực thấp, DriverService xử lý khối lượng ghi lớn (10,000 RPS) hiệu quả. | **Hạn chế tương thích:** gRPC không thân thiện với Browser (cần gRPC-Web proxy cho Web Client). |
| **Tối ưu pin & băng thông:** Tiết kiệm tài nguyên mạng và pin cho ứng dụng tài xế, cải thiện trải nghiệm. | **Yêu cầu kiến thức:** Đội ngũ phát triển cần nắm vững Protobuf và gRPC Streaming. |

---

## 📝 Kết luận
Việc sử dụng **gRPC Client-side Streaming** tối ưu hóa chi phí, hiệu năng và khả năng mở rộng của hệ thống, đáp ứng mục tiêu **Cost Efficiency** và **Hyper-scale** trong giai đoạn phát triển UIT-Go.
