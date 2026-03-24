# AWS Bill Cost Optimization — Best Practice Guide

> A comprehensive, actionable guide to reducing your AWS spend across compute, storage, networking, database, and governance.

---

## Table of Contents

- [Key Metrics](#key-metrics)
- [1. Compute Optimization](#1-compute-optimization)
- [2. Storage Optimization](#2-storage-optimization)
- [3. Networking Optimization](#3-networking-optimization)
- [4. Database Optimization](#4-database-optimization)
- [5. Governance & FinOps](#5-governance--finops)
- [6. AWS Native Tools](#6-aws-native-tools)
- [7. Quick Audit Checklist](#7-quick-audit-checklist)
- [8. Optimization Priority Roadmap](#8-optimization-priority-roadmap)

---

## Key Metrics

| Area | Potential Savings |
|------|------------------|
| EC2 right-sizing | 20–50% on compute |
| Savings Plans / RIs | Up to 72% vs on-demand |
| S3 Lifecycle Policies | 40–60% on storage |
| Spot Instances | Up to 90% vs on-demand |
| Dev/Test scheduling | 60%+ on non-prod environments |
| Idle resources | 10–20% of typical AWS bill |

---

## 1. Compute Optimization

### 1.1 Right-Size EC2 Instances 🔴 High Impact

Over-provisioning is the #1 source of compute waste. Use **AWS Compute Optimizer** to identify instances running below 40% CPU/memory utilization.

**Actions:**
- Run Compute Optimizer across all EC2 instances and Auto Scaling Groups
- Downsize by one instance family tier (e.g., `m5.xlarge` → `m5.large`) = ~50% savings per instance
- Switch to newer generation families (`m4` → `m5` or `m6i`) for better price/performance
- Use `t3`/`t4g` burstable instances for workloads with variable CPU patterns

```bash
# AWS CLI: Get Compute Optimizer EC2 recommendations
aws compute-optimizer get-ec2-instance-recommendations \
  --filters name=Finding,values=Overprovisioned
```

---

### 1.2 Use Savings Plans & Reserved Instances 🔴 High Impact

For predictable, steady-state workloads, commit to discounted pricing.

| Option | Discount | Flexibility |
|--------|----------|-------------|
| Compute Savings Plans (1-yr) | Up to 66% | Applies to EC2, Fargate, Lambda |
| EC2 Instance Savings Plans (1-yr) | Up to 72% | Single instance family + region |
| Reserved Instances (1-yr, No Upfront) | Up to 40% | Specific instance type |
| Reserved Instances (3-yr, All Upfront) | Up to 72% | Specific instance type |

**Recommendation:** Start with **Compute Savings Plans** — they're the most flexible and apply across EC2, Fargate, and Lambda automatically.

```bash
# Check your current Savings Plans coverage
aws ce get-savings-plans-coverage \
  --time-period Start=2025-01-01,End=2025-01-31 \
  --granularity MONTHLY
```

> 💡 **Target:** Aim for >80% Savings Plans coverage on steady workloads before purchasing more on-demand capacity.

---

### 1.3 Use Spot Instances 🔴 High Impact

Spot Instances offer up to **90% discount** over on-demand for interruptible workloads.

**Best use cases:**
- CI/CD pipelines and build agents
- Batch processing and ETL jobs
- Machine learning training
- Stateless web tier with Auto Scaling
- Big data processing (EMR, Spark)

**Best practices:**
- Use **Spot Fleet** with diversified instance pools (multiple instance types + AZs)
- Enable **Spot Instance interruption notices** (2-minute warning) for graceful shutdown
- Use **EC2 Auto Scaling with mixed instances** policy (e.g., 70% Spot / 30% On-Demand)

```json
// Example: Mixed instances policy in Auto Scaling
{
  "MixedInstancesPolicy": {
    "InstancesDistribution": {
      "OnDemandPercentageAboveBaseCapacity": 30,
      "SpotAllocationStrategy": "capacity-optimized"
    }
  }
}
```

---

### 1.4 Schedule Non-Production Environments 🟡 Medium Impact

Dev/test environments running 24/7 are a top source of avoidable spend.

**Approach:**
- Stop EC2 and RDS instances outside business hours (e.g., 8 PM–8 AM weekdays, full weekends)
- Use **AWS Instance Scheduler** (free AWS solution) or EventBridge + Lambda
- Estimated savings: **60–65%** on non-production environment costs

```bash
# Example: Lambda to stop EC2 instances tagged as non-prod
aws ec2 stop-instances \
  --instance-ids $(aws ec2 describe-instances \
    --filters "Name=tag:Environment,Values=dev,staging" \
              "Name=instance-state-name,Values=running" \
    --query "Reservations[].Instances[].InstanceId" \
    --output text)
```

---

### 1.5 Optimize Lambda & Fargate 🟡 Medium Impact

**Lambda:**
- Use **AWS Lambda Power Tuning** (open-source tool) to find the optimal memory/CPU configuration
- Right-sizing memory can reduce both cost and duration simultaneously
- Use **Provisioned Concurrency** only for latency-critical functions — it incurs a continuous charge
- Set appropriate function timeouts to avoid runaway executions

**Fargate:**
- CPU and memory are billed independently — right-size each separately
- Use **Fargate Spot** for fault-tolerant tasks (up to 70% discount)
- Consider migrating long-running Fargate tasks to EC2-backed ECS for better cost control

---

## 2. Storage Optimization

### 2.1 S3 Lifecycle Policies 🔴 High Impact

Implement automated tiering to move objects to cheaper storage classes over time.

**Recommended lifecycle progression:**

```
S3 Standard (0–30 days)
    ↓
S3 Standard-IA (30–90 days)        ~46% cheaper than Standard
    ↓
S3 Glacier Instant Retrieval (90–180 days)  ~68% cheaper than Standard
    ↓
S3 Glacier Deep Archive (180+ days)         ~95% cheaper than Standard
```

**S3 Intelligent-Tiering:** Use for data with unpredictable access patterns — it automatically moves objects between tiers with no retrieval fees.

```json
// Example S3 Lifecycle Policy
{
  "Rules": [{
    "ID": "CostOptimizationRule",
    "Status": "Enabled",
    "Transitions": [
      { "Days": 30,  "StorageClass": "STANDARD_IA" },
      { "Days": 90,  "StorageClass": "GLACIER_IR" },
      { "Days": 180, "StorageClass": "DEEP_ARCHIVE" }
    ],
    "NoncurrentVersionExpiration": { "NoncurrentDays": 30 },
    "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 }
  }]
}
```

> ⚠️ Also enable **abort incomplete multipart uploads** — forgotten multipart uploads silently accumulate storage charges.

---

### 2.2 Right-Size & Clean Up EBS Volumes 🔴 High Impact

EBS volumes continue billing after EC2 instance termination unless explicitly deleted.

**Actions:**
- Identify and delete unattached EBS volumes (`state=available`)
- Migrate `gp2` volumes to `gp3` — same or better performance at ~20% lower cost, no AWS migration required
- Delete old EBS snapshots (use **Amazon Data Lifecycle Manager** to automate retention)
- Use `gp3` as default for all new volumes

```bash
# Find unattached EBS volumes
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query "Volumes[*].{ID:VolumeId,Size:Size,Type:VolumeType}" \
  --output table

# Migrate gp2 to gp3 (in-place, no downtime)
aws ec2 modify-volume --volume-id vol-xxxxxxxx --volume-type gp3
```

---

### 2.3 Use Columnar Formats for Analytics 🟡 Medium Impact

Storing analytics data in Parquet or ORC format instead of CSV/JSON:
- Reduces S3 storage by **60–85%** (compression)
- Reduces Athena query costs by **70–90%** (columnar scanning)
- Improves query performance significantly

---

## 3. Networking Optimization

### 3.1 Reduce NAT Gateway Costs 🔴 High Impact

NAT Gateway charges **$0.045/GB processed** — one of the most common surprise costs on AWS bills.

**Actions:**
- Create **VPC Gateway Endpoints** for S3 and DynamoDB (completely free, routes traffic privately)
- Create **VPC Interface Endpoints** for services accessed frequently (ECR, SSM, Secrets Manager)
- Consolidate to one NAT Gateway per VPC (not per subnet) where HA isn't critical
- Audit NAT Gateway data transfer in Cost Explorer — filter by `NatGateway` service

```bash
# Create a free VPC Gateway Endpoint for S3
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxxxxxxx \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-xxxxxxxx
```

---

### 3.2 Minimize Inter-AZ Data Transfer 🔴 High Impact

Inter-AZ traffic is billed at **$0.01/GB in each direction** and can grow silently to hundreds of dollars per month.

**Actions:**
- Co-locate tightly coupled services in the same AZ for non-HA traffic
- Use **AZ-aware load balancing** — configure ALB and clients to prefer same-AZ targets
- Enable **zonal DNS** for ElastiCache and RDS to reduce cross-AZ reads
- Use **VPC Endpoints** to avoid cross-AZ routing through NAT

---

### 3.3 Use CloudFront for Egress 🟡 Medium Impact

CloudFront data transfer pricing is significantly cheaper than direct EC2/S3 egress.

**Actions:**
- Serve static assets, images, and media via CloudFront
- Put CloudFront in front of API Gateway for global APIs
- Enroll in **CloudFront Security Savings Bundle** — bundles WAF + CloudFront at ~30% discount
- Enable **CloudFront origin shield** to reduce origin requests and bandwidth

---

### 3.4 Audit Elastic IPs & Load Balancers 🟢 Quick Win

- **Elastic IPs** not associated with running instances incur a $0.005/hr charge
- **ALBs/NLBs** bill hourly even when idle
- Consolidate multiple ALBs using host-based and path-based routing rules

```bash
# Find unassociated Elastic IPs
aws ec2 describe-addresses \
  --query "Addresses[?AssociationId==null].PublicIp" \
  --output table
```

---

## 4. Database Optimization

### 4.1 RDS Reserved Instances 🔴 High Impact

RDS Reserved Instances offer up to **69% discount** over on-demand pricing.

| Commitment | Upfront | Discount |
|------------|---------|----------|
| 1-year, No Upfront | $0 | ~40% |
| 1-year, Partial Upfront | Medium | ~42% |
| 3-year, All Upfront | High | ~69% |

> Start with **1-year No Upfront** as a low-risk entry point before committing to 3-year terms.

---

### 4.2 Aurora Serverless v2 for Variable Workloads 🟡 Medium Impact

Aurora Serverless v2 scales between a minimum and maximum ACU range and charges per ACU-hour.

**Use when:**
- Development and test databases
- Applications with infrequent or unpredictable traffic
- Multi-tenant SaaS with variable per-tenant load

**Avoid when:**
- Sustained high-throughput production workloads (provisioned Aurora is cheaper)
- Workloads requiring consistent sub-millisecond latency

---

### 4.3 DynamoDB: On-Demand vs. Provisioned Capacity 🟡 Medium Impact

| Mode | When to Use | Cost |
|------|-------------|------|
| On-Demand | Unpredictable or spiky traffic | ~7× more expensive at steady load |
| Provisioned + Auto Scaling | Predictable or steady traffic | Significantly cheaper |
| DynamoDB Standard-IA Table Class | Infrequently accessed tables | ~60% lower storage cost |

```bash
# Switch a DynamoDB table to provisioned with Auto Scaling
aws dynamodb update-table \
  --table-name MyTable \
  --billing-mode PROVISIONED \
  --provisioned-throughput ReadCapacityUnits=100,WriteCapacityUnits=50
```

---

### 4.4 Add a Caching Layer 🟡 Medium Impact

Deploy **ElastiCache (Redis/Valkey)** in front of RDS/Aurora to reduce read load.

- Even a `cache.t3.micro` node can reduce DB read capacity by 30–50% for read-heavy workloads
- Use **ElastiCache Serverless** for variable cache demand
- Enables downsizing the underlying RDS instance by 1–2 tiers

---

## 5. Governance & FinOps

### 5.1 Enforce Resource Tagging 🔴 Foundation

Without tagging, cost visibility is impossible. Enforce a consistent tagging strategy across all resources.

**Mandatory tags:**

| Tag Key | Example Values | Purpose |
|---------|---------------|---------|
| `Environment` | prod, staging, dev | Separate cost by environment |
| `Team` | backend, data, platform | Chargeback by team |
| `CostCenter` | cc-1234 | Finance allocation |
| `Project` | payments-api | Per-project tracking |
| `Owner` | john.doe@company.com | Accountability |

**Enforce with:**
- AWS Config rule: `required-tags`
- Service Control Policies (SCPs) to prevent resource creation without required tags
- Activate tags as **Cost Allocation Tags** in the Billing Console

---

### 5.2 Set Budgets & Anomaly Alerts 🔴 Foundation

```bash
# Create a monthly budget with 80% alert
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{
    "BudgetName": "Monthly-AWS-Budget",
    "BudgetLimit": {"Amount": "5000", "Unit": "USD"},
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST"
  }' \
  --notifications-with-subscribers '[{
    "Notification": {
      "NotificationType": "ACTUAL",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 80
    },
    "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "ops@company.com"}]
  }]'
```

**Also enable:**
- **AWS Cost Anomaly Detection** — ML-based alerts for unexpected spending spikes
- Separate budgets per team, environment, and major service

---

### 5.3 Implement FinOps Practices 🟡 Process

- Hold **monthly cost reviews** with engineering leads
- Track **unit economics**: cost per user, cost per API request, cost per GB processed
- Include cost estimates in architecture review processes
- Create team-level cost dashboards using AWS Cost Explorer or a BI tool (Grafana, QuickSight)
- Make engineers responsible for the costs of the services they own

---

### 5.4 AWS Organizations Best Practices 🟡 Governance

- Enable **consolidated billing** for volume discounts across all accounts
- Use **Service Control Policies (SCPs)** to:
  - Block creation of expensive instance types in dev accounts
  - Restrict deployment to approved regions only
  - Prevent disabling of Cost Allocation Tags
- Use separate accounts per environment (prod/staging/dev) for cost isolation

---

## 6. AWS Native Tools

| Tool | Purpose | Cost |
|------|---------|------|
| **AWS Cost Explorer** | Visualize, filter, and forecast spend | Free |
| **AWS Compute Optimizer** | Right-sizing recommendations for EC2, Lambda, EBS | Free (Enhanced Recommendations = paid) |
| **AWS Trusted Advisor** | Automated cost, security, and performance checks | Business/Enterprise Support required for full access |
| **Cost Anomaly Detection** | ML-based anomaly alerts | Free (pay for SNS/actions) |
| **AWS Budgets** | Spend/usage budgets with automated alerts | First 2 budgets free, then $0.02/day |
| **S3 Storage Lens** | Organization-wide S3 analytics | Free dashboard; advanced metrics are paid |
| **AWS Pricing Calculator** | Estimate costs before deploying | Free |
| **Lambda Power Tuning** | Optimal Lambda memory configuration | Open-source, runs in your account |
| **AWS Instance Scheduler** | Automated start/stop of EC2 and RDS | Free (deploy via CloudFormation) |

---

## 7. Quick Audit Checklist

Run this audit monthly or before each quarterly planning cycle.

### Immediate Wins (< 1 day effort)

- [ ] Identify top 5 services by spend in Cost Explorer
- [ ] List all unattached EBS volumes and delete them
- [ ] Release all unassociated Elastic IPs
- [ ] Check for idle/unused load balancers
- [ ] Delete old EBS snapshots older than 90 days
- [ ] Run AWS Trusted Advisor cost checks

### Short-Term Actions (1–2 weeks)

- [ ] Review Savings Plans coverage report — target >80%
- [ ] Enable Cost Anomaly Detection for all services
- [ ] Audit NAT Gateway data processed cost and add VPC Endpoints
- [ ] Check for `gp2` EBS volumes and migrate to `gp3`
- [ ] Enable S3 Lifecycle Policies on top 10 buckets by size
- [ ] Validate all resources have required cost allocation tags

### Strategic Initiatives (1–4 weeks)

- [ ] Run Compute Optimizer for all EC2 instances and implement recommendations
- [ ] Purchase Compute Savings Plans based on stable baseline usage
- [ ] Set up AWS Budgets per team/project with 80% and 100% alerts
- [ ] Schedule dev/test environment shutdowns
- [ ] Migrate eligible workloads to Spot Instances

---

## 8. Optimization Priority Roadmap

| Priority | Action | Estimated Savings | Effort |
|----------|--------|-------------------|--------|
| 1 | Purchase Compute Savings Plans | 30–66% on compute | Low |
| 2 | Right-size EC2 with Compute Optimizer | 20–50% per instance | Medium |
| 3 | Implement S3 Lifecycle Policies | 40–60% on S3 | Low |
| 4 | Add VPC Gateway Endpoints (S3, DynamoDB) | Eliminate NAT costs | Low |
| 5 | Schedule dev/test shutdowns | 60%+ on non-prod | Medium |
| 6 | Migrate gp2 → gp3 EBS volumes | ~20% on EBS | Low |
| 7 | Enable Cost Anomaly Detection + Budgets | Prevent overruns | Low |
| 8 | RDS Reserved Instances | Up to 69% on RDS | Low |
| 9 | Migrate eligible EC2 to Spot | Up to 90% on those instances | High |
| 10 | Add caching layer (ElastiCache) | 30–50% on DB reads | Medium |

---

## Additional Resources

- [AWS Cost Optimization Hub](https://aws.amazon.com/aws-cost-management/aws-cost-optimization/)
- [AWS Well-Architected Framework — Cost Optimization Pillar](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html)
- [FinOps Foundation](https://www.finops.org/)
- [AWS Pricing Calculator](https://calculator.aws/)
- [Lambda Power Tuning (GitHub)](https://github.com/alexcasalboni/aws-lambda-power-tuning)
- [AWS Instance Scheduler](https://aws.amazon.com/solutions/implementations/instance-scheduler-on-aws/)

---

*Last updated: March 2026 | Covers AWS pricing as of 2025–2026*
