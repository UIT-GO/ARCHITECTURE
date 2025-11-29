# ADR 007: Chuyển Đổi Giao Thức Real-time — Từ WebSocket sang gRPC Streaming
**Trạng thái:** Đã chấp nhận

---

## 📌 Bối cảnh
- Hệ thống UIT-Go ban đầu sử dụng **WebSocket** cho các luồng Real-time như cập nhật vị trí tài xế và đẩy thông báo.
- Công ty bước vào giai đoạn **Hyper-growth**, dự kiến **tăng trưởng 20 lần** và cần chịu tải **10,000 RPS**.
- KPI cốt lõi: **Cost Efficiency** — yêu cầu giảm chi phí hạ tầng từ **$0.04 → $0.01 / đơn hàng**.
- Cần một giao thức Real-time **thống nhất**, **tối ưu băng thông**, **hiệu suất cao hơn WebSocket** ở quy mô lớn.

---

## 🧩 Quyết định
**Chuyển đổi toàn bộ giao tiếp Real-time** giữa Client ↔ Backend  
từ **WebSocket** sang **gRPC Streaming**:

- Client-side Streaming  
- Server-side Streaming  
- Bidirectional Streaming  

---

## 🔍 Các lựa chọn đã cân nhắc

| Lựa chọn | Ưu điểm | Nhược điểm (Lý do bị thay thế) |
|---------|---------|---------------------------------|
| **Tiếp tục dùng WebSocket** | Đơn giản, phổ biến, tương thích tốt với browser | Không tối ưu chi phí; dùng JSON/Text nặng hơn Protobuf → tăng băng thông & chi phí ở Hyper-scale |
| **Chuyển sang gRPC Streaming** | Protobuf nhị phân siêu nhỏ gọn; dùng HTTP/2; hiệu suất vượt trội; giao thức thống nhất | Chi phí refactor; Web Client cần proxy gRPC-Web; phức tạp hơn WebSocket |

---

## ⚖️ Đánh đổi (Trade-offs)

### ✅ Lợi ích Đạt được (Đáp ứng KPI)
- **Cost Efficiency tối đa**  
  - Protobuf nhị phân tiết kiệm băng thông (chỉ vài bytes mỗi bản ghi vị trí) → giảm chi phí/network load.
- **Scalability vượt trội**  
  - gRPC Streaming trên HTTP/2 chịu tải tốt hơn rất nhiều khi cần truyền dữ liệu liên tục (10,000 RPS).
- **Thống nhất kiến trúc**  
  - Cùng một giao thức real-time cho Mobile, Backend, và Microservices → đơn giản hóa vận hành & giám sát.
- **Hiệu suất cao**  
  - Overhead thấp, latency thấp, tiết kiệm pin & CPU cho client.

### ⚠️ Chi phí & Rủi ro Chấp nhận
- **Chi phí refactor**  
  - Cần viết lại phần lớn logic Client/Server từ WebSocket → gRPC Streaming.
- **Phức tạp hạ tầng**  
  - API Gateway cần hỗ trợ gRPC.  
  - Web Client cần dùng **gRPC-Web proxy**.
- **Hạn chế tương thích cho Web**  
  - WebSocket đơn giản hơn cho Frontend; chuyển sang gRPC-Web có thể làm chậm tốc độ dev ban đầu.

---

## 📝 Kết luận
Việc chuyển đổi từ **WebSocket → gRPC Streaming** là **quyết định chiến lược** giúp UIT-Go chuyển mình từ giai đoạn **MVP** sang **Hyper-scale**.

Mặc dù có chi phí refactor và thay đổi hạ tầng, lợi ích về:

- **Cost Efficiency**  
- **Performance**  
- **Scalability**  
- **Tối ưu băng thông/tài nguyên**

vượt xa chi phí bỏ ra, đồng thời đảm bảo đáp ứng các KPI tài chính và kỹ thuật quan trọng của công ty trong giai đoạn tăng trưởng mạnh.

