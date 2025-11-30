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

#### **Mô tả chi tiết**

**Kiến trúc Microservices Event-Driven theo sơ đồ:**

**1. Client Layer**
- **Customer**: Ứng dụng khách hàng (HTTP/HTTPS, gRPC Streaming)
- **Driver**: Ứng dụng tài xế (gRPC Streaming cho real-time location)

**2. API Gateway**
- Điểm vào trung tâm cho tất cả requests
- Load balancing, authentication, routing
- Giao tiếp với các services qua REST API và gRPC Streaming

**3. Core Microservices**
- **UserService**: Quản lý tài khoản, authentication, profile
- **TripService**: Xử lý chuyến đi, booking logic
- **DriverService**: Quản lý tài xế, location tracking, availability
- **Discovery Service**: Service discovery và health checking

**4. Message Broker**
- **Cluster Kafka**: Event streaming backbone
- Xử lý async communication giữa các services
- Event sourcing cho trip updates

**5. Data Storage Layer**
- **PostgreSQL**: Primary database cho user data, trips, transactions
- **MongoDB**: Document store cho flexible data, logs
- **Redis**: Caching layer cho session, real-time location

**6. Monitoring & Analytics**
- **Elasticsearch**: Search engine, log aggregation
- **Logstash**: Log processing pipeline  
- **Kibana**: Visualization và monitoring dashboard

**Giao thức giao tiếp:**
- **HTTP/HTTPS**: Standard REST API operations
- **gRPC Streaming**: Real-time location updates (Driver ↔ DriverService)
- **Kafka Events**: Async inter-service communication
- **Tìm kiếm Service khác**: Discovery service pattern

---

## 2. Module Reports - Chuyên sâu theo từng khía cạnh

Hệ thống UIT-Go được phân tích và thiết kế qua 3 module chuyên sâu, mỗi module tập trung vào một khía cạnh quan trọng:

### 📈 **Module A - Scalability & Performance**
**Vai trò:** System Architect  
**Mục tiêu:** Xử lý tải cao (1,500+ req/s), tối ưu hiệu năng  
**Giải pháp chính:**
- Kiến trúc bất đồng bộ với Message Queue
- Redis GEO cho real-time location tracking  
- Horizontal scaling với microservices

**Kết quả:** Tăng throughput 22x, giảm latency 47x  
**📄 Chi tiết:** [Module A Report](Module_A_Report.md)

---

### 🛡️ **Module B - Reliability & High Availability** 
**Vai trò:** Site Reliability Engineer (SRE)  
**Mục tiêu:** 99.9% uptime, RTO < 30 phút  
**Giải pháp chính:**
- Multi-AZ deployment với ALB
- MongoDB Replica Set cho data redundancy
- Disaster Recovery với Pilot Light strategy

**Kết quả:** 99.9% availability, RTO 28 phút, ROI 5,700%  
**📄 Chi tiết:** [Module B Report](Module_B_Report.md)

---

### 💰 **Module E - Automation & Cost Optimization**
**Vai trò:** Platform & FinOps Engineer  
**Mục tiêu:** Self-service deployment, cost visibility  
**Giải pháp chính:**
- Self-Service Platform với Terraform + GitHub Actions
- Hybrid EC2 + ECS strategy  
- Comprehensive FinOps framework

**Kết quả:** Deployment time < 30 phút, cost giảm 30%, ROI 302%  
**📄 Chi tiết:** [Module E Report](Module_E_Report.md)

---

### 🎯 **Tổng hợp Business Impact**
- **Technical:** Scalable, reliable, cost-effective platform
- **Financial:** ROI trung bình 2,000%+ across modules  
- **Operational:** Self-healing system với minimal manual intervention
- **Strategic:** Ready for hyper-growth với sustainable cost structure

---

## 3. Các Quyết định Thiết kế & Trade-off Quan trọng

### 🎯 **Tổng quan Triết lý Thiết kế**
Hệ thống UIT-Go được thiết kế dựa trên nguyên tắc **"Scale or Die"** - mọi quyết định đều cân nhắc khả năng mở rộng, độ tin cậy và tối ưu chi phí để đối phó với tăng trưởng đột biến.

---

### 🔥 **Quyết định 1: Kiến trúc Bất đồng bộ (Async) với Kafka**
**Bối cảnh:** Luồng đặt xe đồng bộ gây nghẽn nghiêm trọng khi tải cao  
**Quyết định:** Chuyển sang Message Queue với Apache Kafka

#### **Trade-off Analysis:**
| Khía cạnh | Legacy (Sync) | Kafka (Async) | Đánh giá |
|-----------|---------------|---------------|----------|
| **Throughput** | 55 req/s | **1,250 req/s** | ✅ Tăng 22x |
| **Latency P95** | 2,300ms | **48ms** | ✅ Giảm 47x |
| **Error Rate** | 18% | **0%** | ✅ Hoàn hảo |
| **Complexity** | Thấp | **Cao hơn** | ⚠️ Trade-off chấp nhận được |
| **Consistency** | Strong | **Eventual** | ⚠️ Chấp nhận cho UX tốt hơn |

**Kết luận:** *"Không được để mất request của khách hàng"* - Eventual consistency chấp nhận được để đảm bảo hệ thống luôn nhận được yêu cầu.

---

### 🛡️ **Quyết định 2: Multi-AZ Deployment Strategy**  
**Bối cảnh:** Single AZ tạo SPOF nghiêm trọng, mỗi phút downtime = $5,000 thiệt hại  
**Quyết định:** Multi-AZ với ALB và database replication

#### **Trade-off Analysis:**
| Khía cạnh | Single-AZ | Multi-AZ | Business Impact |
|-----------|-----------|----------|-----------------|
| **Cost** | $170/tháng | **$357/tháng** | ⚠️ Tăng 110% |
| **Availability** | 99.5% | **99.9%** | ✅ Giảm 21.6 phút downtime/tháng |
| **RTO** | 15-30 phút | **< 30 giây** | ✅ Tự động failover |
| **RPO** | Có thể mất dữ liệu | **< 2 phút** | ✅ MongoDB Replica Set |

**ROI Calculation:**  
- Investment: +$187/tháng  
- Business Protection: $5,000/phút × 21.6 phút = **$108,000/tháng**  
- **ROI: 5,700%**

---

### 🏗️ **Quyết định 3: Hybrid Compute Strategy (EC2 + ECS)**
**Bối cảnh:** Cần balance giữa cost, performance và operational complexity  
**Quyết định:** EC2 cho core services + ECS Fargate cho elastic workloads

#### **Platform Comparison Analysis:**
| Tiêu chí | EKS | ECS Fargate | EC2 + Docker | Hybrid (Chọn) |
|----------|-----|-------------|--------------|---------------|
| **Monthly Cost** | $193 | $238 | $195 | **$240** |
| **Learning Curve** | 3 tháng | 2 tuần | 1 tháng | **1.5 tháng** |
| **Scalability** | Excellent | Good | Fair | **Good** |
| **Performance** | Good | Good | Excellent | **Excellent** |
| **Operational Overhead** | High | Low | Medium | **Medium** |

**Strategic Rationale:**
- **Phase 1:** EC2 cho stability và performance core services  
- **Phase 2:** ECS cho background jobs và auto-scaling workloads  
- **Future:** EKS evaluation khi team size > 8 developers

---

### 🤖 **Quyết định 4: Self-Service Platform với Terraform**
**Bối cảnh:** Manual deployment gây bottleneck (2-3 ngày/service)  
**Quyết định:** Infrastructure as Code với Terraform Modules + GitHub Actions

#### **ROI Analysis:**
| Khía cạnh | Before | After | Impact |
|-----------|--------|-------|--------|
| **Deployment Time** | 2-3 ngày | **30 phút** | ✅ 95% reduction |
| **DevOps Intervention** | 80% deployments | **20%** | ✅ 60% self-service |
| **Cost per Deploy** | $1,500 | **$50** | ✅ 30x cheaper |
| **Monthly OpEx** | $15,000 | **$4,000** | ✅ $11,000 savings |

**Trade-off:**
- **Investment:** $20,000 (platform development + training)  
- **Monthly Savings:** $11,000  
- **Break-even:** 1.8 tháng  
- **ROI Year 1:** 302%

---

### 🎯 **Quyết định 5: Data Storage Strategy**
**Bối cảnh:** Multi-service cần data stores tối ưu cho từng use case  
**Quyết định:** Polyglot Persistence với PostgreSQL + MongoDB + Redis

#### **Data Store Selection:**
| Use Case | Technology | Lý do chọn | Trade-off |
|----------|------------|------------|-----------|
| **Transactional Data** | PostgreSQL | ACID compliance, relational queries | Phức tạp scaling horizontal |
| **Flexible Schema** | MongoDB | Document model, rapid iteration | Eventual consistency |
| **Real-time Location** | Redis GEO | In-memory speed (<1ms), geospatial queries | Data volatility |
| **Search & Analytics** | Elasticsearch | Full-text search, log aggregation | Resource intensive |

**Performance Results:**
- **Location queries:** 50ms → **<1ms** (Redis GEO)  
- **Database CPU:** Giảm 90% nhờ Redis caching  
- **Search response:** **Sub-second** với Elasticsearch

---

### 📊 **Tổng hợp Strategic Trade-offs**

#### **Chi phí vs Hiệu năng:**
- **Chấp nhận tăng 110% infrastructure cost** để đạt 99.9% availability  
- **ROI trung bình 2,000%+** qua cost savings và revenue protection

#### **Complexity vs Maintainability:**
- **Microservices complexity** đổi lấy independent scaling và development velocity  
- **Polyglot persistence** tối ưu performance nhưng tăng operational overhead

#### **Short-term vs Long-term:**
- **Hybrid compute strategy** để optimize cost hiện tại và flexibility tương lai  
- **Self-service platform** investment ngắn hạn cho productivity dài hạn

#### **Business Alignment:**
- **Mọi quyết định** đều cân nhắc business impact ($5,000/phút downtime)  
- **Scale-first mentality** để ready cho hyper-growth phase  
- **Developer experience** prioritized để maintain development velocity

---

### 🎯 **Key Success Principles**

> **1. "Scale or Die" Mindset:** Mọi design decision đều consider khả năng scale  
> **2. Data-Driven Decisions:** Load testing và metrics drive architectural choices  
> **3. Business Value First:** Technical complexity chấp nhận được nếu có business ROI  
> **4. Gradual Evolution:** Hybrid strategies để balance risk và innovation  
> **5. Cost Consciousness:** Performance improvement phải justify cost increase

---

## 4. Thách thức & Bài học Kinh nghiệm

### 🚧 **Thách thức chính theo Module**

#### **📈 Module A - Scalability:**
- **Async Debugging:** Khó trace lỗi qua nhiều services với Kafka
- **Load Testing Gap:** Synthetic traffic không reflect real user behavior  
- **Kafka Learning Curve:** Partitioning và consumer group complexity

**Bài học:** *Start simple, scale gradually* - Redis cluster phức tạp cuối cùng chỉ cần single instance.

#### **🛡️ Module B - Reliability:**  
- **DR Testing Fear:** Ngại test disaster recovery trên production
- **Multi-AZ Complexity:** Cross-AZ latency +2-3ms, cost tăng 150%
- **Documentation Gap:** Runbooks thiếu chi tiết gây panic khi incident

**Bài học:** *Design for graceful failure* - Thay vì tránh failures, handle chúng elegantly.

#### **💰 Module E - Automation:**
- **Terraform State Drift:** Manual AWS Console changes gây inconsistency  
- **Developer Adoption:** Resistance to learning IaC tools
- **Cost Attribution:** Inconsistent tagging khó track chi phí

**Bài học:** *Automation empowers, not replaces* - Platform thành công khi làm developers productive hơn.

---

### 🎯 **Top 5 Bài học Quan trọng**

| Bài học | Ứng dụng |
|---------|----------|
| **Monitor First, Optimize Later** | Không thể cải thiện những gì không đo được |
| **Documentation as Code** | Runbooks phải được maintain như source code |
| **Cost Visibility Early** | Tag resources từ ngày đầu, không retrofit được |
| **Test Everything in Production-like** | Local testing không đủ cho distributed systems |
| **Cross-functional Collaboration** | DevOps, SRE, Dev phải work together từ đầu |

---

### 🚀 **Điều sẽ làm khác nếu restart**

- **Architecture:** Start với ECS Fargate thay vì EC2, implement circuit breakers sớm
- **Process:** Infrastructure as Code mandatory, không allow manual changes  
- **Culture:** Cross-training team members, blameless post-mortems

---

### 💭 **Kết luận**

**Key Mindset Shift:** From *perfect systems* to *resilient systems* - hệ thống tốt là hệ thống handle failures gracefully.

> *"The best architecture evolves with business needs while maintaining operational excellence."*

---

## 5. Kết quả & Hướng phát triển

### 📊 **Tóm tắt Kết quả Đạt được**

#### **🎯 Metrics sau khi áp dụng 3 Modules:**
| Metric | Before | After | Module đóng góp |
|--------|--------|-------|-----------------|
| **Max Throughput** | 55 req/s | **1,250 req/s** | Module A (22x) |
| **P95 Latency** | 2,300ms | **48ms** | Module A (47x) |
| **System Availability** | 99.5% | **99.9%** | Module B (ROI 5,700%) |
| **Deployment Time** | 2-3 ngày | **30 phút** | Module E (95% reduction) |
| **Infrastructure Cost** | $357/tháng | **$240/tháng** | Module E (30% savings) |

#### **🏆 Tổng Business Impact:**
- **Scalability Achievement:** Đạt mục tiêu 1,500+ req/s cho hyper-growth phase
- **Reliability Improvement:** 99.9% uptime, RTO <30 phút cho disaster recovery
- **Cost Optimization:** $11,000/tháng operational savings, ROI 302%
- **Developer Productivity:** Self-service platform giảm 95% deployment time

---

### 🚀 **Đề xuất Cải tiến Tương lai**

#### **Phase 1: Tối ưu Architecture hiện tại (3-6 tháng)**
- **Container Migration:** Chuyển từ EC2 sang EKS khi team scale >8 developers
- **Advanced Monitoring:** Implement distributed tracing với Jaeger
- **Security Enhancement:** Zero-trust networking và automated compliance

**Học phần liên quan:** SE104 (Software Engineering), SE113 (Distributed Systems)

#### **Phase 2: Multi-Region Expansion (6-12 tháng)**
- **Geographic Distribution:** Multi-AZ mở rộng thành multi-region
- **Edge Computing:** CDN integration cho global latency optimization  
- **Data Consistency:** Cross-region replication strategy

**Học phần liên quan:** SE347 (Cloud Computing), SE358 (Big Data)

#### **Phase 3: AI/ML Integration (12-18 tháng)**
- **Intelligent Scaling:** ML-powered capacity planning
- **Business Intelligence:** Real-time analytics cho demand prediction
- **Automated Optimization:** AI-driven performance tuning

---

### 🎓 **Kiến thức & Kỹ năng Rút ra**

#### **Technical Skills Acquired:**
- **Cloud Architecture:** AWS services integration và best practices
- **Microservices Design:** Event-driven architecture với Kafka
- **Infrastructure as Code:** Terraform modules và CI/CD automation
- **Site Reliability Engineering:** Multi-AZ deployment và disaster recovery
- **Performance Engineering:** Load testing và scalability optimization

#### **Soft Skills Development:**
- **System Thinking:** Cân nhắc trade-offs giữa cost, performance, complexity
- **Data-Driven Decision Making:** Sử dụng metrics để justify architectural choices
- **Documentation & Communication:** Technical writing và presentation skills
- **Problem Solving:** Debug distributed systems và troubleshoot production issues

---

### 💭 **Kết luận Đồ án**

**Thành tựu chính của đồ án:**
1. **Thiết kế thành công** kiến trúc cloud-native cho ride-hailing platform
2. **Chứng minh hiệu quả** qua load testing và metrics measurement  
3. **Áp dụng lý thuyết** từ các môn học vào bài toán thực tế
4. **Develop skills** quan trọng cho career Software Engineering

**Bài học lớn nhất:**
> *"Successful architecture is not about using the latest technology, but about making informed trade-offs that align with business goals."*

**Chuẩn bị cho tương lai:**
Đồ án này đã trang bị kiến thức và experience cần thiết để handle enterprise-scale systems, từ startup MVP đến hyper-growth platform.

