# AWS Fundamentals Workshop — Opening Slides

## EC2 and S3 for Data Engineering Teams

---

### Slide 1: Title

> **AWS Fundamentals Workshop**
> EC2 and S3 for Data Engineering Teams
>
> Getting Started with AWS Cloud Infrastructure
>
> Presented by: [Your Name]
> Date: [Workshop Date]
> Duration: 1 Hour

---

### Slide 2: Introductions

**About This Workshop**

- Your Instructor: [Name, Title, Experience]

**Who Should Attend:**

- Data Engineers new to AWS
- Teams transitioning to cloud infrastructure
- Anyone working with data storage and compute in AWS

**Prerequisites:**

- Basic understanding of cloud concepts
- Familiarity with command line interfaces
- No prior AWS experience required

**What You'll Need:**

- AWS account access
- Laptop with SSH client (or browser for SSM)

---

### Slide 3: Workshop Agenda — Today's Schedule (60 Minutes)

**Part 1: Amazon EC2 — Compute Foundation (30 minutes)**

- Create IAM roles for secure access
- Configure security groups for network isolation
- Launch and connect to EC2 instances (SSH + SSM)
- Install AWS CLI and verify access

**Part 2: Amazon S3 — Storage and Access Control (30 minutes)**

- Create S3 buckets for data storage
- Look up your SSO role ARN (your existing identity — no new users needed)
- Implement bucket policies with explicit Deny
- Demonstrate layered access control (IAM policy vs bucket policy)
- Test SSO-based and role-based access patterns

**Bonus:**

- IAM Roles Anywhere for on-premises and dev workstations

---

### Slide 4: What We're Building Today

```
┌─────────────────────────────────────────────────────┐
│                    AWS Cloud                         │
│                                                     │
│  ┌──────────────────┐         ┌─────────────────┐  │
│  │   EC2 Instance   │         │   S3 Bucket     │  │
│  │                  │         │                 │  │
│  │  • IAM Role      │────────▶│  • Raw Data     │  │
│  │  • AWS CLI       │  Upload │  • Processed    │  │
│  │  • Security SG   │◀────────│  • Archive      │  │
│  └──────────────────┘ Download└─────────────────┘  │
│         ▲                            ▲              │
│         │                            │              │
│    ┌────┴─────┐              ┌───────┴────────┐    │
│    │ SSH/SSM  │              │ IAM User/Group │    │
│    └──────────┘              └────────────────┘    │
│         ▲                            ▲              │
└─────────┼────────────────────────────┼──────────────┘
          │                            │
    ┌─────┴──────┐            ┌────────┴───────┐
    │   You!     │            │  On-Prem /     │
    │ (Internal  │            │  Dev Box       │
    │  Network)  │            │ (Roles         │
    └────────────┘            │  Anywhere)     │
                              └────────────────┘
```

**Key Components:**

| Component | Purpose |
|-----------|---------|
| EC2 Instance | Virtual server for data processing |
| IAM Role | Secure credentials without access keys |
| S3 Bucket | Scalable object storage for data |
| Security Group | Network firewall (internal access only) |
| Bucket Policy | Fine-grained access control with explicit Deny |
| IAM Roles Anywhere | Extend AWS access to on-prem systems |

---

### Slide 5: Why This Matters

**Security Best Practices**

- ✅ No hardcoded credentials in code or config files
- ✅ Temporary credentials that rotate automatically
- ✅ Network isolation with security groups
- ✅ Principle of least privilege with IAM policies
- ✅ Explicit Deny in bucket policies for defense in depth

**Operational Excellence**

- ✅ Consistent access patterns across environments
- ✅ Audit trail of all API calls via CloudTrail
- ✅ Scalable storage without capacity planning
- ✅ Multiple connection methods (SSH, SSM)

**Cost Optimization**

- ✅ Pay only for what you use
- ✅ Stop instances when not needed
- ✅ S3 lifecycle policies for automatic archival
- ✅ Right-sized compute resources

**Foundation for Advanced Patterns**

- Data lakes and analytics pipelines
- ETL workflows with AWS Glue
- Big data processing with EMR
- Machine learning with SageMaker

---

### Slide 6: Learning Objectives

**By the end of this workshop, you will:**

**Understand AWS Security Model**

- How IAM roles provide temporary credentials
- Difference between IAM policies and bucket policies
- Why explicit Deny always wins
- How security groups control network access

**Deploy Core Infrastructure**

- Launch EC2 instances with proper IAM roles
- Create S3 buckets with encryption enabled
- Configure security groups for internal access
- Connect using both SSH and SSM

**Implement Access Control**

- Look up your SSO role ARN (your existing identity)
- Write bucket policies with Allow and Deny statements
- Demonstrate access denial and approval flows
- Understand how multiple policies interact

**Apply Best Practices**

- Use IAM roles instead of access keys
- Implement least-privilege access
- Use IAM Roles Anywhere for on-premises workloads
- Follow AWS Well-Architected principles

---

> **Let's Get Started!**
> Questions before we dive in?
