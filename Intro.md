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





