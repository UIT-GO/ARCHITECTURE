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
- **Technical:** Microservices architecture với event-driven communication
- **Operational:** Infrastructure as Code với automated deployment pipelines  
- **Reliability:** Multi-AZ deployment với fault tolerance mechanisms
- **Strategic:** Cloud-native design patterns cho sustainable scaling

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
| **Throughput** | Giới hạn bởi blocking I/O | **Cao hơn đáng kể** | ✅ Non-blocking architecture |
| **Latency** | Cao do cascading calls | **Thấp hơn nhiều** | ✅ Decoupled processing |
| **Error Rate** | Cao khi overload | **Gần như zero** | ✅ Queue buffering |
| **Complexity** | Đơn giản | **Phức tạp hơn** | ⚠️ Distributed system complexity |
| **Consistency** | Strong | **Eventual** | ⚠️ CAP theorem trade-off |

**Kết luận:** *"Không được để mất request của khách hàng"* - Eventual consistency chấp nhận được để đảm bảo hệ thống luôn nhận được yêu cầu.

---

### 🛡️ **Quyết định 2: Multi-AZ Deployment Strategy**  
**Bối cảnh:** Single AZ tạo SPOF nghiêm trọng, mỗi phút downtime = $5,000 thiệt hại  
**Quyết định:** Multi-AZ với ALB và database replication

#### **Trade-off Analysis:**
| Khía cạnh | Single-AZ | Multi-AZ | Architectural Impact |
|-----------|-----------|----------|----------------------|
| **Cost** | Thấp | **Cao hơn đáng kể** | ⚠️ Redundancy overhead |
| **Availability** | Có SPOF | **Fault tolerant** | ✅ Eliminate single points of failure |
| **RTO** | Phụ thuộc manual recovery | **Tự động failover** | ✅ Automated disaster recovery |
| **RPO** | Risk mất dữ liệu | **Data replication** | ✅ Continuous backup strategy |

**Design Principle:**  
- **Availability over Cost:** Chấp nhận chi phí cao hơn để đạt fault tolerance  
- **Automated Recovery:** Giảm thiểu human intervention trong disaster scenarios  
- **Geographic Distribution:** Isolation failure domains

---

### 🏗️ **Quyết định 3: Hybrid Compute Strategy (EC2 + ECS)**
**Bối cảnh:** Cần balance giữa cost, performance và operational complexity  
**Quyết định:** EC2 cho core services + ECS Fargate cho elastic workloads

#### **Architectural Pattern Analysis:**
| Tiêu chí | Container Orchestration | Serverless Containers | Virtual Machines | Hybrid Approach |
|----------|-------------------------|---------------------|------------------|------------------|
| **Resource Efficiency** | Tối ưu density | Pay-per-use model | Predictable cost | **Balanced approach** |
| **Learning Curve** | Steep | Moderate | Familiar | **Gradual adoption** |
| **Scalability Pattern** | Horizontal auto-scaling | Event-driven scaling | Manual/script scaling | **Selective optimization** |
| **Performance** | Container overhead | Cold start latency | Native performance | **Best of both worlds** |
| **Operational Model** | DevOps intensive | Managed service | Infrastructure management | **Layered complexity** |

**Strategic Rationale:**
- **Phase 1:** EC2 cho stability và performance core services  
- **Phase 2:** ECS cho background jobs và auto-scaling workloads  
- **Future:** EKS evaluation khi team size > 8 developers

---

### 🤖 **Quyết định 4: Self-Service Platform với Terraform**
**Bối cảnh:** Manual deployment gây bottleneck (2-3 ngày/service)  
**Quyết định:** Infrastructure as Code với Terraform Modules + GitHub Actions

#### **Automation Benefits Analysis:**
| Khía cạnh | Manual Process | Automated Pipeline | Architectural Benefit |
|-----------|----------------|-------------------|----------------------|
| **Deployment Velocity** | Slow, error-prone | **Consistent & fast** | ✅ Continuous delivery principles |
| **Human Dependency** | High manual intervention | **Self-service model** | ✅ Developer empowerment |
| **Process Consistency** | Variable outcomes | **Standardized workflows** | ✅ Infrastructure as Code |
| **Operational Efficiency** | Labor intensive | **Automated pipelines** | ✅ DevOps culture adoption |

**Design Trade-off:**
- **Initial Complexity:** Platform development overhead  
- **Long-term Benefits:** Sustainable development velocity  
- **Cultural Shift:** From manual to automated operations  
- **Scalability Foundation:** Self-service infrastructure

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

**Architectural Benefits:**
- **Spatial Queries:** In-memory geospatial indexing pattern  
- **Database Load:** Cache-aside pattern giảm database pressure  
- **Search Performance:** Full-text indexing với distributed search

---

### 📊 **Tổng hợp Strategic Trade-offs**

- **Chi phí vs Hiệu năng:**
- **Trade-off acceptable cost increase** cho fault tolerance và high availability  
- **Long-term value creation** through operational efficiency và risk mitigation

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
- **DR Testing Fear:** Ngại test disaster recovery trên production environment
- **Multi-AZ Complexity:** Cross-AZ network overhead và cost implications
- **Documentation Gap:** Runbooks thiếu chi tiết gây operational challenges

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

- **Architecture:** Prioritize managed services, implement resilience patterns từ đầu
- **Process:** Infrastructure as Code first approach, immutable deployments  
- **Culture:** DevOps collaboration, continuous learning mindset

---

### 💭 **Kết luận**

**Key Mindset Shift:** From *perfect systems* to *resilient systems* - hệ thống tốt là hệ thống handle failures gracefully.

> *"The best architecture evolves with business needs while maintaining operational excellence."*

---

## 5. Kết quả & Hướng phát triển

### 📊 **Tóm tắt Kết quả Đạt được**

#### **🎯 Architectural Achievements theo 3 Modules:**
| Aspect | Legacy System | Modern Architecture | Module đóng góp |
|--------|---------------|-------------------|------------------|
| **Scalability** | Monolithic bottlenecks | **Event-driven microservices** | Module A (Async patterns) |
| **Response Time** | Synchronous blocking | **Non-blocking processing** | Module A (Queue decoupling) |
| **Reliability** | Single points of failure | **Fault-tolerant design** | Module B (Multi-AZ strategy) |
| **Deployment** | Manual, error-prone | **Automated CI/CD** | Module E (IaC pipelines) |
| **Cost Efficiency** | Over-provisioned resources | **Right-sized infrastructure** | Module E (Cloud optimization) |

#### **🏆 Architectural Design Success:**
- **Scalability Achievement:** Event-driven architecture support hyper-growth scenarios
- **Reliability Improvement:** Multi-AZ deployment với automated failover mechanisms
- **Cost Optimization:** Cloud-native patterns với resource optimization strategies
- **Developer Productivity:** Self-service platform theo DevOps best practices

---

### 🚀 **Đề xuất Cải tiến Tương lai**

#### **Phase 1: Architecture Maturation**
- **Container Orchestration:** Evolution to container-native platforms
- **Observability Enhancement:** Distributed tracing và monitoring strategies
- **Security Hardening:** Zero-trust principles và compliance automation

**Học phần liên quan:** SE104 (Software Engineering), SE113 (Distributed Systems)

#### **Phase 2: Geographic Distribution**
- **Multi-Region Architecture:** Cross-region deployment patterns
- **Edge Computing:** Content delivery và latency optimization  
- **Data Consistency:** Distributed data management strategies

**Học phần liên quan:** SE347 (Cloud Computing), SE358 (Big Data)

#### **Phase 3: Intelligent Systems**
- **Predictive Scaling:** Machine learning cho capacity management
- **Analytics Integration:** Real-time data processing pipelines
- **Optimization Automation:** AI-driven performance improvement

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

