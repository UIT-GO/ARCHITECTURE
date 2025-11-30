# BÁO CÁO CHUYÊN SÂU ĐỒ ÁN UIT-GO  
**Xây dựng Nền tảng Cloud-Native cho Ride-hailing Service**

**Nhóm:** UIT-Go Team  
**Thành viên:**  
- Võ Minh Kiệt – 22520727  
- Võ Mai Nguyên – 22520991  

---

## 1. Tổng quan kiến trúc hệ thống

### 1.1. Bối cảnh Business
UIT-Go đang trong giai đoạn **"Scale or Die"** với sự tăng trưởng đột biến từ **2,500 CCU** lên **50,000 CCU** trong chiến dịch Marketing *"Đồng giá 5k"*. Hệ thống cũ gặp hai thách thức nghiêm trọng:

- **Chi phí:** Tăng từ $0.04/giao dịch lên $40,000/tháng  
- **Scalability:** Quá tải khi traffic > 500 req/s, mất doanh thu $100/phút

### 1.2. Sơ đồ kiến trúc tổng thể

![Sơ đồ kiến trúc tổng quan](Image/BASIC.png)

**Kiến trúc Multi-AZ High Availability:**
```
                    Internet Gateway
                           │
                    ┌──────▼──────┐
                    │     ALB     │
                    │(Cross-AZ)   │
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                 │                  │
   ┌────▼────┐       ┌────▼────┐       ┌────▼────┐
   │   AZ-1a │       │   AZ-1b │       │   AZ-1c │
   │ EC2 Pod │       │ EC2 Pod │       │ EC2 Pod │
   └─────────┘       └─────────┘       └─────────┘
        │                 │                  │
   ┌────▼────────────────▼──────────────────▼────┐
   │        MongoDB Replica Set (3 nodes)        │
   │     Redis Cluster + Kafka Cluster           │
   └──────────────────────────────────────────────┘
```

### 1.3. Thành phần kiến trúc chi tiết

| Thành phần | Công nghệ | Vai trò | SLA Target |
|------------|-----------|---------|------------|
| **API Gateway** | AWS ALB / Nginx | Entry point, routing, security | 99.9% availability |
| **Discovery Service** | Consul / Eureka | Service registration & discovery | - |
| **Auth Service** | Spring Boot + PostgreSQL | User management, JWT authentication | < 100ms response |
| **Trip Service** | Spring Boot + MongoDB | Trip lifecycle, orchestration | < 200ms response |
| **Driver Service** | Spring Boot + Redis GEO | Location tracking, driver matching | < 50ms geospatial query |
| **Message Queue** | Kafka / Amazon SQS | Async communication, event streaming | 99.99% message delivery |
| **Load Balancer** | AWS ALB | Traffic distribution, health checks | Auto-failover < 10s |
| **Database** | MongoDB Replica Set + PostgreSQL | Data persistence, ACID compliance | RPO < 5 min, RTO < 30 min |

---

## 2. Phân tích Module chuyên sâu

### 2.1. Module A – Scalability & Performance

#### **Vấn đề cốt lõi:**
Hệ thống legacy không thể đáp ứng yêu cầu tăng trưởng về throughput và latency.

#### **Architectural Patterns áp dụng:**
1. **Asynchronous Processing Pattern:**
   - Tách biệt request handling và business processing
   - Sử dụng message queue làm buffer
   - Immediate response với eventual consistency

2. **CQRS (Command Query Responsibility Segregation):**
   - Tối ưu read operations với in-memory cache
   - Write operations qua persistent storage
   - Geospatial queries với specialized data store

#### **Trade-offs:**
- **Gained:** Throughput cao, latency thấp, fault tolerance
- **Lost:** Data consistency ngay lập tức, complexity tăng

---

### 2.2. Module B – Reliability & High Availability

#### **Reliability Principles:**
- **Redundancy:** Loại bỏ single points of failure
- **Fault Isolation:** Service failures không cascade
- **Graceful Degradation:** System vẫn hoạt động khi có component down

#### **HA Patterns:**
1. **Multi-AZ Deployment:** Geographic distribution cho availability
2. **Database Replication:** Primary-secondary với automatic failover
3. **Circuit Breaker:** Prevent cascading failures
4. **Health Checks:** Proactive failure detection

#### **DR Strategy:**
- **Pilot Light Pattern:** Minimal infrastructure luôn sẵn sàng
- **Backup & Recovery:** Automated với RTO/RPO targets
- **Chaos Engineering:** Định kỳ test failure scenarios

---

### 2.3. Module E – Automation & Cost Optimization

#### **FinOps Principles:**
- **Cost Visibility:** Detailed cost allocation per service
- **Right-sizing:** Resource optimization dựa trên usage patterns
- **Automation:** Giảm manual operations và human errors

#### **Self-Service Platform:**
- **Infrastructure as Code:** Standardized, reproducible deployments
- **GitOps:** Version-controlled infrastructure changes
- **Cost Governance:** Automated cost controls và alerts

#### **Hybrid Compute Strategy:**
- **EC2:** Cho stable, long-running services
- **ECS Fargate:** Cho elastic, event-driven workloads
- **Cost Trade-off:** Flexibility vs simplicity

---

## 3. Tổng hợp quyết định thiết kế & Trade-off

### 3.1. Architecture Pattern: Microservices vs Monolith

#### **Rationale cho Microservices:**
- **Scalability:** Independent scaling per service
- **Team Autonomy:** Parallel development và deployment
- **Technology Diversity:** Right tool for right job
- **Fault Isolation:** Service failures không lan truyền

#### **Trade-offs:**
- **Complexity tăng:** Distributed system challenges
- **Network overhead:** Inter-service communication latency
- **Data consistency:** Eventual consistency thay vì ACID
- **Operational overhead:** Monitoring, logging phức tạp hơn

---

### 3.2. Communication Pattern: Sync vs Async

#### **Hybrid Approach:**
- **Synchronous:** User-facing operations cần immediate response
- **Asynchronous:** Internal workflows có thể eventual consistency

#### **Design Principles:**
- **Loose Coupling:** Services giao tiếp qua events
- **Resilience:** Message queues làm buffer
- **Scalability:** Async processing tăng throughput

---

### 3.3. Data Strategy: Polyglot Persistence

#### **Database Selection Rationale:**
- **PostgreSQL:** ACID compliance cho user data
- **MongoDB:** Document flexibility cho trip data
- **Redis:** Sub-millisecond geospatial queries

#### **Trade-offs:**
- **Performance optimization** vs **Operational complexity**
- **Technology flexibility** vs **Team expertise requirements**

---

### 3.4. Availability Strategy: Multi-AZ vs Single AZ

#### **Multi-AZ Active-Active Pattern:**
- **Redundancy:** Eliminate single points of failure
- **Geographic distribution:** Protect against AZ outages
- **Load distribution:** Better resource utilization

#### **Cost-Benefit Analysis:**
High availability investment justified bởi business impact của downtime.

---

### 3.5. Compute Platform: Hybrid Strategy

#### **EC2 + ECS Hybrid Approach:**
- **EC2:** Stable, long-running core services
- **ECS Fargate:** Elastic, event-driven workloads
- **Future path:** Migration to EKS khi team scale

#### **Decision Framework:**
Cân bằng giữa cost efficiency, operational simplicity, và team capabilities.

---

## 4. Thách thức & Bài học kinh nghiệm

### 4.1. Distributed Systems Challenges

#### **Observability & Debugging:**
- **Vấn đề:** Multi-service request flows khó trace và debug
- **Pattern:** Correlation ID để track requests cross-services
- **Lesson:** Centralized logging và distributed tracing là essential

#### **Data Consistency:**
- **Challenge:** Eventual consistency trong microservices
- **Approach:** Saga pattern cho distributed transactions
- **Trade-off:** Performance vs consistency guarantees

---

### 4.2. Cost Management Lessons

#### **Premature Optimization Anti-Pattern:**
- **Problem:** Over-provisioning dựa trên assumptions
- **Solution:** Data-driven optimization với monitoring
- **Principle:** "Measure first, optimize second"

#### **Resource Right-Sizing:**
- **Approach:** Continuous monitoring và adjustment
- **Automation:** Cost alerts và automated scaling policies

---

### 4.3. Reliability Engineering

#### **Chaos Engineering Insights:**
- **Practice:** Regular failure testing để verify resilience
- **Discovery:** Real-world failures khác với theoretical scenarios
- **Improvement:** Iterative tuning dựa trên test results

#### **DR Testing Reality:**
- **Gap:** Difference between planned vs actual recovery times
- **Solution:** Regular drills và continuous improvement

---

### 4.4. Organizational Learning

#### **Technology Adoption:**
- **Pattern:** Initial productivity dip, then significant gains
- **Mitigation:** Structured learning programs và pair programming
- **Investment:** Training time pays off trong long-term productivity

#### **Automation Trade-offs:**
- **Benefit:** Reduced manual errors, increased velocity
- **Risk:** Over-automation without proper guardrails
- **Balance:** Automation với appropriate safety controls

---

## 5. Kết quả & Hướng phát triển

### 5.1. Key Achievements

#### **Scalability & Performance:**
- Throughput improvement: Significant increase trong system capacity
- Latency reduction: Substantial improvement trong response times
- Reliability enhancement: Near-zero error rates under normal load

#### **Reliability & Availability:**
- High availability: Target uptime objectives achieved
- Recovery objectives: RTO và RPO targets met successfully
- Fault tolerance: Automatic failover mechanisms functioning

#### **Automation & Cost Optimization:**
- Developer velocity: Major reduction trong deployment times
- Cost efficiency: Significant infrastructure cost savings
- Self-service adoption: Majority of operations automated

---

### 5.2. Architecture Evolution Roadmap

#### **Near-term (6-12 months):**
- **Observability Enhancement:** Advanced monitoring và tracing capabilities
- **Security Hardening:** Enhanced authentication và authorization mechanisms
- **Performance Tuning:** Fine-tuning based on production metrics

#### **Medium-term (1-2 years):**
- **Container Orchestration:** Evaluate migration to Kubernetes
- **Database Optimization:** Sharding và performance improvements
- **Multi-region:** Geographic distribution cho global scale

#### **Long-term (2+ years):**
- **AI/ML Integration:** Intelligent scaling và cost optimization
- **Serverless Migration:** Event-driven architecture expansion
- **Enterprise Features:** Compliance và advanced governance capabilities

---

### 5.3. Success Measurement Framework

#### **Technical Metrics:**
- **Performance:** Throughput, latency, error rates monitoring
- **Reliability:** Availability, recovery times, failure rates tracking
- **Efficiency:** Resource utilization, cost per transaction analysis

#### **Business Metrics:**
- **Developer Productivity:** Deployment frequency, time-to-market measurement
- **Operational Efficiency:** Support tickets, manual interventions reduction
- **Cost Management:** Infrastructure spend optimization, ROI tracking

---

**Tài liệu tham khảo:**
- `ARCHITECTURE.md`, `Module_A_Report.md`, `Module_B_Report.md`, `Module_E_Report.md`, thư mục `ADR/`.
