# AWS Workshop Quick Reference — EC2 & S3 Fundamentals

**One-Page Summary**

---

## Part 1: EC2 Setup (30 min)

| Step | Time | Action |
|------|------|--------|
| 1 | 5 min | **Create IAM Role** — IAM Console → Roles → Create role → EC2 → Attach `AmazonSSMManagedInstanceCore` + `AmazonS3FullAccess` → Name: `Workshop-EC2-S3-SSM-Role` → Copy Role ARN |
| 2 | 3 min | **Create Key Pair** — EC2 Console → Key Pairs → Name: `workshop-keypair` → RSA / .pem → Download → `chmod 400` |
| 3 | 3 min | **Create Security Group** — Name: `Workshop-Internal-Access-SG` → SSH from `YOUR-INTERNAL-CIDR-1` + `YOUR-INTERNAL-CIDR-2` → All traffic from same |
| 4 | 5 min | **Launch EC2** — Name: `workshop-data-eng-demo` → Amazon Linux 2023 → t3.micro → Attach key pair, SG, IAM role → Auto-assign public IP: **Disable** → private subnet |
| 5 | 5 min | **Connect via SSH** — `ssh -i workshop-keypair.pem ec2-user@PRIVATE-IP` (must be on internal network/VPN) |
| 6 | 3 min | **Connect via SSM** — EC2 Console → Select instance → Connect → Session Manager (preferred — works without VPN) |
| 7 | 6 min | **Test AWS CLI** — `aws --version` → `aws sts get-caller-identity` → `aws s3 ls` → Create test file |

## Part 2: S3 Access Control (30 min)

| Step | Time | Action |
|------|------|--------|
| 8 | 3 min | **Look Up SSO Role ARN** — Open CloudShell from console nav bar → `aws sts get-caller-identity` → copy the base role ARN from output |
| 9 | 3 min | **Create S3 Bucket** — Name: `workshop-demo-bucket-[initials]-[number]` → Same region as EC2 → Defaults |
| 10 | 7 min | **Apply Bucket Policy** — Allow only your SSO role → Deny all others via `StringNotLike` condition |
| 11 | 5 min | **Test EC2 Access** — ❌ Should fail (explicit Deny blocks the role) |
| 12 | 5 min | **Test SSO Access via CloudShell** — ✅ Open CloudShell from console nav → `aws s3 ls s3://$BUCKET_NAME/` (no profile needed — uses your logged-in identity) |
| 13 | 5 min | **Update Bucket Policy** — Add EC2 role to Allow + Deny exception list |
| 14 | 5 min | **Verify EC2 Access** — ✅ Now works |

---

## Quick Commands

```bash
# SSH
ssh -i workshop-keypair.pem ec2-user@PUBLIC-IP

# Check identity (run on laptop to get your SSO role ARN)
aws sts get-caller-identity
aws sts get-caller-identity --profile your-sso-profile

# S3 operations
aws s3 ls                                    # List buckets
aws s3 ls s3://bucket-name/                  # List contents
aws s3 cp file.txt s3://bucket-name/         # Upload
aws s3 cp s3://bucket-name/file.txt ./       # Download

# SSO access
aws sso login --profile your-sso-profile
aws s3 ls s3://bucket-name/ --profile your-sso-profile
```

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| SSH fails | Check security group, instance state, key permissions (`chmod 400`) |
| SSM fails | Verify IAM role attached, wait 5 min for registration |
| S3 Access Denied | Check bucket policy Principal list and Deny exception list |
| Policy not working | Validate JSON syntax, wait 1–2 min, verify saved |

---

## Key Concepts

**Access Control Layers:**

- **IAM Policy:** What can this identity do?
- **Bucket Policy:** Who can access this bucket?
- **Explicit Deny always wins** — even over IAM Allow

**Security Groups:** `YOUR-INTERNAL-CIDR-1`, `YOUR-INTERNAL-CIDR-2`

**ARN Format:** `arn:aws:iam::ACCOUNT-ID:role/RoleName`

---

## Bonus: IAM Roles Anywhere

Use X.509 certificates from your own CA to grant on-premises workloads, dev boxes, and CI/CD pipelines temporary AWS credentials — no long-lived access keys needed.

1. Register your CA as a trust anchor
2. Create a profile mapped to an IAM role
3. Install the credential helper on your machine
4. Your workload gets temporary credentials automatically

---

## Cleanup

```bash
aws ec2 terminate-instances --instance-ids i-XXXXX
aws s3 rm s3://bucket-name/ --recursive
aws s3 rb s3://bucket-name
# Delete IAM role, security group, key pair via Console
```
