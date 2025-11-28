# 📜 Kịch bản Tài chính & Kinh doanh cho UIT-Go

## 1. Bối cảnh Công ty (The Context)

**Giai đoạn:** Startup Series A — đang trong thời điểm sống còn *"Scale or Die"*.

**Tình hình:** UIT-Go vừa nhận được **$500,000 đầu tư**. Nhà đầu tư yêu cầu công ty phải **chiếm lĩnh thị trường xe ôm công nghệ TP.HCM trong 6 tháng**.

**Sự kiện sắp diễn ra:** Chiến dịch Marketing **“Đồng giá 5k”** sẽ được tung ra vào tháng sau.  
👉 *Dự kiến: số lượng người dùng tăng gấp **20 lần** (Hyper-growth).*

---

## 2. Bài toán Tài chính (The Financial Problem)

Hệ thống hiện tại chỉ phù hợp cho giai đoạn nhỏ lẻ (50.000 user), không phù hợp cho hyper-scale.

### **Chi phí hiện tại**
- Tổng chi phí hạ tầng: **$2,000/tháng**
- 50.000 user → **$0.04 / đơn hàng**
- Kiến trúc: Monolith / Basic Microservices

### **Khủng hoảng sắp tới**
Khi user tăng lên **1,000,000** sau chiến dịch:

- Chi phí tăng tuyến tính → **$40,000/tháng**
- Runway cạn kiệt chỉ sau **3 tháng**
- Hệ thống dễ sập khi peak traffic  
  👉 **Downtime gây thiệt hại $5,000/giờ**

➡️ *Nếu không nâng cấp kiến trúc → công ty phá sản.*

---

## 3. Mục tiêu cho Đội Kỹ thuật (Module A Goals)

CTO giao KPI cụ thể cho nhóm Kiến trúc (System Architects):

### **1. Scalability**
- Hệ thống phải chịu được: **10,000 Requests/giây (RPS)**
- Không được phép downtime trong chiến dịch Marketing

### **2. Cost Efficiency**
- Giảm chi phí hạ tầng từ **$0.04 → $0.01 / đơn hàng**
- Dù user tăng 20 lần, tiền cloud chỉ được tăng tối đa **5 lần**
  → ~ **$10,000/tháng**

---

# 💡 Cách đưa Kịch bản này vào Báo cáo (Rất thuyết phục)

Dưới đây là 3 ví dụ mẫu để bạn dùng trực tiếp trong REPORT.md và Slide.

---

## ✨ 1. Tại sao dùng Redis (Caching)?

> **Lập luận tài chính + kỹ thuật**

Việc truy vấn trực tiếp vào PostgreSQL tiêu tốn rất nhiều CPU. Để đạt hiệu năng mong muốn, chúng ta cần nâng cấp database lên dòng **db.r5.4xlarge** với chi phí khoảng **$2,000/tháng**.  
Thay vào đó, nhóm sử dụng **Redis Cluster** làm caching layer.

- Chi phí Redis: **~$200/tháng**
- Giảm tải cho DB: **~90%**
- Tiết kiệm: **$1,800/tháng**

➡️ *Quyết định sử dụng Redis vừa tăng tốc độ vừa phù hợp bài toán tài chính khi công ty bước vào Hyper-growth.*

---

## ✨ 2. Tại sao dùng Message Queue (Async Architecture)?

> **Lập luận chống downtime**

Vào giờ cao điểm, mỗi phút hệ thống sập gây thiệt hại:

- **~$100 doanh thu**
- **~500 khách hàng rời bỏ** chuyển sang đối thủ

Việc áp dụng **SQS** giúp đảm bảo **Zero Downtime**, mặc dù traffic tăng đột biến.

- Chi phí SQS: vài USD cho hàng triệu request
- Lợi ích: chống mất dữ liệu, chống overloaded, tự phục hồi

➡️ *Message Queue chính là “bảo hiểm chống sập hệ thống” với chi phí cực thấp.*

---

## ✨ 3. Tại sao chọn Read Replicas?

> **Lập luận về khả năng mở rộng DB**

Khi user tăng gấp 20 lần, một database duy nhất sẽ trở thành điểm nghẽn vật lý.

Read Replica giúp:

- Tăng khả năng scale luồng đọc lên gấp nhiều lần
- Chi phí thấp hơn Sharding rất nhiều
- Dễ mở rộng trong 1–3 tháng tới mà không gián đoạn dịch vụ

➡️ *Đây là bước bắt buộc trong giai đoạn “Scale or Die”.*

---

# 🎯 Kết luận

Kịch bản trên là nền tảng để:

- Giải thích vì sao bạn chọn Multi-AZ  
- Vì sao có Load Balancer  
- Vì sao nhân đôi service  
- Vì sao dùng Redis, Queue, Replica…  
- Và vì sao đây là kiến trúc **bền vững – tiết kiệm – chịu tải cao**

Đưa kịch bản này vào bài sẽ biến bài của bạn thành **một dự án thực chiến** chứ không còn là “bài tập kỹ thuật đơn thuần”.

