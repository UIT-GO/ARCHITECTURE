# ADR 002: Lựa chọn Compute Platform - EKS vs ECS vs EC2

## Trạng thái (Status)
**Accepted (Implementation Phase)**
*Lưu ý: Quyết định này được đưa ra để tối ưu hóa trade-off giữa chi phí, hiệu năng và công sức vận hành cho hệ thống UIT-Go.*

## 1. Bối cảnh (Context)

### Thách thức Compute Platform
Với kiến trúc Multi-AZ đã triển khai thành công, UIT-Go cần quyết định compute platform cho việc mở rộng hệ thống:

#### Yêu cầu hiện tại:
- **Scalability:** Hỗ trợ từ 3 services hiện tại lên 10+ services trong 6 tháng tới
- **Cost Optimization:** Giảm compute cost từ $357/tháng xuống dưới $250/tháng
- **Operational Efficiency:** Platform & FinOps Engineer có thể quản lý toàn bộ infrastructure
- **Developer Experience:** Deployment time < 10 phút, rollback < 5 phút

#### Business Context:
- **Team Size:** 1 Platform Engineer, 4 Developers
- **Service Growth:** Auth, Trip, Driver services → 10 microservices
- **Traffic Pattern:** 1,500 req/s peak, 200 req/s average
- **Budget Constraint:** Total infrastructure budget $500/tháng

## 2. Phân tích Lựa chọn (Alternatives Analysis)

### Option A: Amazon EKS (Kubernetes)
#### Chi phí Analysis:
- **Control Plane:** $73/tháng (fixed)
- **Worker Nodes:** 3x t3.medium = $95/tháng  
- **Add-ons:** ALB Controller, EBS CSI = $10/tháng
- **Data Transfer:** ~$15/tháng
- **Total:** **$193/tháng**

#### Ưu điểm:
- **Container-native:** Best practices cho microservices
- **Auto-scaling:** HPA + VPA cho resource optimization
- **Service Mesh Ready:** Istio/Linkerd integration
- **Vendor Neutral:** Portable across cloud providers

#### Nhược điểm:
- **Operational Complexity:** Kubernetes learning curve 2-3 tháng
- **Fixed Cost:** $73/tháng control plane bất kể usage
- **Debugging:** Pod networking, service discovery complexity
- **Security:** Additional attack surface, RBAC management

#### Performance:
- **Startup Time:** 30-60s (container + k8s overhead)
- **Resource Efficiency:** 85-90% (container density)
- **Networking:** Service mesh latency +2-5ms

---

### Option B: Amazon ECS (Fargate)
#### Chi phí Analysis:
- **Fargate Compute:** $0.04048/vCPU-hour, $0.004445/GB-hour
- **3 services, 2 tasks each:** 6 vCPU + 12GB RAM
- **Monthly Cost:** 6×730×$0.04048 + 12×730×$0.004445 = **$216/tháng**
- **ALB:** $22/tháng
- **Total:** **$238/tháng**

#### Ưu điểm:
- **Serverless:** No server management overhead
- **Pay-per-use:** Scale to zero, cost theo actual usage
- **AWS Native:** Tight integration với AWS services
- **Simple Operations:** Minimal operational complexity

#### Nhược điểm:
- **Cold Start:** 10-30s startup time
- **Vendor Lock-in:** AWS specific, không portable
- **Limited Customization:** Restricted kernel access, networking
- **Cost Scaling:** Expensive cho consistent workloads

#### Performance:
- **Startup Time:** 10-30s (cold start)
- **Resource Efficiency:** 95% (managed by AWS)
- **Networking:** VPC mode, minimal overhead

---

### Option C: EC2 với Docker Compose [CURRENT]
#### Chi phí Analysis:
- **Compute:** 2x t3.large = $150/tháng
- **EBS:** 2x 20GB GP3 = $8/tháng
- **ALB:** $22/tháng
- **Data Transfer:** $15/tháng
- **Total:** **$195/tháng**

#### Ưu điểm:
- **Cost Predictable:** Fixed monthly cost
- **Full Control:** Complete OS và hardware access
- **Performance:** Bare metal performance, no virtualization overhead
- **Simplicity:** Docker Compose dễ hiểu và debug

#### Nhược điểm:
- **Manual Scaling:** Cần scripting cho auto-scaling
- **Server Management:** OS patching, security updates
- **Limited Orchestration:** No service discovery, health management
- **Operational Overhead:** SSH access, log management

#### Performance:
- **Startup Time:** 5-10s (docker container)
- **Resource Efficiency:** 70-80% (over-provisioning for peaks)
- **Networking:** Direct host networking, lowest latency

## 3. Trade-off Analysis Matrix

| Criteria | EKS Weight (30%) | ECS Weight (25%) | EC2 Weight (45%) |
|----------|------------------|------------------|------------------|
| **Monthly Cost** | $193 | $238 | $195 |
| **Operational Complexity** | High (8/10) | Low (3/10) | Medium (5/10) |
| **Scalability** | Excellent (9/10) | Good (7/10) | Fair (5/10) |
| **Performance** | Good (7/10) | Good (7/10) | Excellent (9/10) |
| **Learning Curve** | High (3 months) | Low (2 weeks) | Medium (1 month) |
| **Vendor Lock-in** | Low | High | Medium |

### Weighted Score Calculation:
- **EKS:** (193×0.4) + (8×0.2) + (9×0.2) + (7×0.2) = **125.6 points**
- **ECS:** (238×0.4) + (3×0.2) + (7×0.2) + (7×0.2) = **98.6 points**
- **EC2:** (195×0.4) + (5×0.2) + (5×0.2) + (9×0.2) = **96.8 points**

*Lưu ý: Cost được tính ngược (lower is better), Operational Complexity càng cao càng tệ*

## 4. Quyết định (Decision)

**CHỌN HYBRID APPROACH: EC2 + ECS Tasks cho specific workloads**

### Rationale:
1. **Cost Optimization:** Giữ EC2 cho base workload ($195/tháng)
2. **Selective ECS:** Sử dụng ECS Fargate cho:
   - Background jobs (intermittent)
   - Auto-scaling workloads (peak traffic)
   - Development/staging environments

### Implementation Strategy:
- **Phase 1:** Continue EC2 cho core services (Auth, Trip, Driver)
- **Phase 2:** Migrate background processing sang ECS Fargate
- **Phase 3:** Evaluate EKS khi team size tăng lên 8+ developers

## 5. Chi tiết Cost Breakdown (Hybrid Approach)

### Current State (EC2 Only):
```
Core Services (EC2):           $195/month
├── 2x t3.large instances:     $150
├── EBS storage:               $8
├── ALB:                       $22
└── Data transfer:             $15
```

### Target State (Hybrid):
```
Base Infrastructure (EC2):     $195/month
├── Core services:             $195
Elastic Workloads (ECS):       $45/month
├── Background jobs:           $25
├── Staging environments:      $15
└── Auto-scaling tasks:        $5
TOTAL:                         $240/month
```

### Cost Projection (6 months):
- **Month 1-2:** $195 (EC2 only)
- **Month 3-4:** $220 (partial ECS migration)
- **Month 5-6:** $240 (full hybrid)
- **Savings vs pure ECS:** $238 - $240 = **Break-even with flexibility**

## 6. Performance Impact Analysis

### Latency Comparison:
| Metric | EC2 | ECS | EKS |
|--------|-----|-----|-----|
| **Cold Start** | 5-10s | 10-30s | 30-60s |
| **Network Latency** | 0.1ms | 0.5ms | 2-5ms |
| **CPU Performance** | 100% | 95% | 85-90% |
| **Memory Efficiency** | 80% | 95% | 90% |

### Scalability Metrics:
- **EC2:** Manual scaling, 5-10 phút
- **ECS:** Auto-scaling, 2-5 phút  
- **EKS:** HPA scaling, 30s-2 phút

## 7. Operational Impact

### Team Skill Requirements:
| Platform | Learning Time | Ongoing Maintenance | Debugging Complexity |
|----------|---------------|-------------------|---------------------|
| **EC2** | 1 month | 4 hours/week | Medium |
| **ECS** | 2 weeks | 2 hours/week | Low |
| **EKS** | 3 months | 8 hours/week | High |

### Maintenance Overhead:
- **EC2:** OS patching, security updates, capacity planning
- **ECS:** Service definitions, task placement strategies  
- **EKS:** Cluster upgrades, node management, addon updates

## 8. Risk Assessment

### Technical Risks:
| Risk | EC2 | ECS | EKS |
|------|-----|-----|-----|
| **Vendor Lock-in** | Low | High | Medium |
| **Skill Dependency** | Medium | Low | High |
| **Operational Failure** | High | Low | Medium |
| **Cost Overrun** | Low | Medium | High |

### Mitigation Strategies:
- **EC2:** Automated backup, infrastructure as code
- **ECS:** Cost monitoring, resource right-sizing
- **Hybrid:** Gradual migration, rollback capabilities

## 9. Success Metrics

### Cost Metrics:
- **Target:** <$250/tháng total compute cost
- **Efficiency:** >70% resource utilization
- **Growth:** Cost per service <$25/tháng

### Performance Metrics:
- **Latency:** P95 <200ms maintained
- **Availability:** >99.9% uptime
- **Scalability:** Handle 2x traffic với cùng cost

### Operational Metrics:
- **Deployment Time:** <10 phút
- **MTTR:** <30 phút
- **Team Productivity:** 1 Platform Engineer manage 10+ services

## 10. Implementation Roadmap

### Phase 1 (Month 1-2): Optimization Current EC2
- [ ] Right-size instances based on metrics
- [ ] Implement auto-scaling scripts
- [ ] Optimize Docker images và startup time

### Phase 2 (Month 3-4): Selective ECS Migration  
- [ ] Migrate background jobs sang ECS Fargate
- [ ] Setup cost monitoring và alerting
- [ ] Performance comparison testing

### Phase 3 (Month 5-6): Hybrid Operations
- [ ] Establish hybrid deployment patterns
- [ ] Cost optimization recommendations
- [ ] Prepare for future EKS evaluation

**Decision Review:** 6 tháng sau implementation để evaluate hiệu quả và consider EKS migration nếu team scale up.