# ADR 001: Sử dụng Terraform Modules và GitHub Actions cho Self-Service Platform

## Trạng thái (Status)
**Accepted (Implementation Phase)**
*Lưu ý: Quyết định này được triển khai để xây dựng nền tảng tự phục vụ cho developers, nhằm giảm Time-to-Market và tối ưu hóa chi phí vận hành.*

## 1. Bối cảnh (Context)
Trong giai đoạn "Scale or Die" của UIT-Go, việc triển khai hạ tầng và services mới đang trở thành bottleneck nghiêm trọng:

### Thách thức hiện tại:
1. **Manual Deployment Process:** Mỗi lần deploy service mới cần 2-3 ngày làm việc của DevOps team
2. **Inconsistent Infrastructure:** Mỗi service được setup khác nhau, gây khó khăn trong monitoring và cost allocation
3. **High Operational Cost:** DevOps team phải can thiệp vào 80% các deployment, gây nghẽn cổ chai
4. **Cost Visibility Gap:** Không thể theo dõi chi phí theo từng service/team, dẫn đến waste tài nguyên

### Business Impact:
- **Time-to-Market chậm:** Feature mới cần 1 tuần để production
- **Developer Experience kém:** Dev phải chờ DevOps setup môi trường
- **Chi phí vận hành cao:** $15,000/tháng cho DevOps team, nhưng chỉ có 3 services

## 2. Quyết định (Decision)
Chúng tôi quyết định xây dựng **Self-Service Platform** dựa trên:

* **Infrastructure as Code:** Terraform Modules tái sử dụng được
* **CI/CD Automation:** GitHub Actions với approval workflow
* **Cost Management:** Automated tagging và AWS Cost Explorer integration
* **Developer Experience:** One-click deployment với safety guardrails

### Kiến trúc Platform:
- **Terraform Modules:** Standardized templates cho microservices, databases, monitoring
- **GitHub Actions Workflows:** Automated deployment pipelines với cost estimation
- **Environment Structure:** Dev/Staging/Production với consistent configuration
- **Cost Allocation:** Automatic tagging strategy cho resource tracking

## 3. Phân tích Lựa chọn (Alternatives Analysis)

### Option A: Kubernetes-based Platform (ArgoCD + Helm)
* **Ưu điểm:** Container-native, GitOps workflow, strong ecosystem
* **Nhược điểm:** Learning curve cao, cần EKS cluster ($2,400/năm), phức tạp cho team nhỏ
* **Kết luận:** LOẠI BỎ vì cost/benefit không hợp lý cho giai đoạn hiện tại

### Option B: AWS Service Catalog
* **Ưu điểm:** Native AWS integration, built-in approval workflow
* **Nhược điểm:** Vendor lock-in, limited customization, UI/UX kém
* **Kết luận:** LOẠI BỎ vì không flexible cho yêu cầu tùy chỉnh

### Option C: Terraform + GitHub Actions [ĐƯỢC CHỌN]
* **Ưu điểm:** 
  - Open source, flexible customization
  - Strong community support
  - GitHub native integration
  - Cost-effective (miễn phí cho GitHub Actions public repos)
* **Nhược điểm:** Requires Terraform expertise
* **Kết luận:** **CHẤP NHẬN** vì balance tốt giữa flexibility và cost

## 4. Thiết kế Chi tiết (Design Specification)

### 4.1. Terraform Module Strategy
- **Microservice Module:** Standardized EC2/ALB/ASG configuration với best practices
- **Database Module:** RDS/MongoDB setup với backup và monitoring tự động
- **Monitoring Module:** CloudWatch alerts và dashboards cho mỗi service
- **Tagging Strategy:** Consistent tags cho cost allocation và resource management

### 4.2. CI/CD Pipeline Design
- **Cost Estimation:** Tự động tính toán chi phí trước khi deploy
- **Approval Workflow:** Production deployments cần approval nếu cost > threshold
- **Safety Guardrails:** Automated checks cho security và cost limits
- **Rollback Mechanism:** Tự động rollback nếu deployment fail

### 4.3. Cost Management Framework
- **Automated Tagging:** Service, Team, Environment, Cost Center tags
- **Budget Alerts:** AWS Budgets cho mỗi service với threshold warnings  
- **Cost Reporting:** Weekly cost analysis và optimization recommendations
- **Resource Rightsizing:** Automated suggestions cho instance optimization

## 5. Hệ quả & Đánh đổi (Consequences & Trade-offs)

| Tiêu chí | Phân tích tác động |
|----------|-------------------|
| **Developer Velocity** | **Tích cực:** Giảm deployment time từ 2-3 ngày xuống 30 phút. Developer tự deploy được mà không cần DevOps |
| **Cost Visibility** | **Tích cực:** 100% tài nguyên được tag consistent, có thể track cost theo service/team real-time |
| **Infrastructure Consistency** | **Tích cực:** Standardized modules đảm bảo best practices và security |
| **Learning Curve** | **Tiêu cực:** Developer cần học Terraform basics (ước tính 1 tuần training) |
| **Platform Maintenance** | **Tiêu cực:** Cần maintain modules và CI/CD pipelines (0.5 FTE) |

### ROI Analysis:
- **Investment:** 
  - Platform development: 160 giờ ($16,000)
  - Developer training: 40 giờ ($4,000)
  - **Tổng:** $20,000
- **Savings:**
  - DevOps efficiency: $8,000/tháng
  - Cost optimization: $3,000/tháng  
  - **Break-even:** 1.8 tháng

## 6. Success Metrics & Monitoring

### 6.1. Platform Adoption Metrics
- **Deployment Frequency:** Target 10+ deploys/tuần
- **Mean Time to Deploy:** Target < 30 phút
- **Self-Service Adoption:** Target 80% deployments không cần DevOps

### 6.2. Cost Optimization Metrics  
- **Cost per Service:** Track monthly spend per microservice
- **Resource Utilization:** Target >70% CPU/Memory utilization
- **Waste Reduction:** Target 20% cost savings trong Q1

### 6.3. Developer Experience Metrics
- **Time to First Deploy:** Measure onboarding efficiency cho new developers
- **Error Rate:** Track deployment success rate
- **Support Tickets:** Monitor reduction in DevOps intervention requests

## 7. Implementation Roadmap

### Phase 1 (Week 1-2): Foundation
- Tạo Terraform modules cơ bản (microservice, database)
- Setup GitHub Actions workflows
- Implement cost tagging strategy

### Phase 2 (Week 3-4): Advanced Features  
- Cost estimation integration (Infracost)
- AWS Budget automation  
- Monitoring dashboard setup

### Phase 3 (Week 5-6): Developer Experience
- Documentation và training materials
- CLI tool cho common tasks
- Cost optimization recommendations engine

**Completion Target:** 6 tuần
**Resource Requirement:** 1 Platform Engineer (full-time)

## 8. Risk Mitigation

### 8.1. Technical Risks
- **Terraform State Management:** Sử dụng remote state với S3 + DynamoDB locking
- **Security Compliance:** Built-in security scanning trong CI/CD pipeline
- **Cost Overruns:** Automated cost limits và approval workflows

### 8.2. Organizational Risks  
- **Adoption Resistance:** Comprehensive training và phased rollout
- **Knowledge Concentration:** Documentation và cross-training cho team
- **Platform Maintenance:** Dedicated platform team và community support model