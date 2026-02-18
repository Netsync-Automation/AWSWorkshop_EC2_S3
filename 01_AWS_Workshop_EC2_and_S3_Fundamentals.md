# AWS Workshop: EC2 and S3 Fundamentals

**1-Hour Hands-On Session for Data Engineering Teams**

---

## Workshop Agenda (60 minutes)

### Part 1: EC2 Instance Setup (30 minutes)

1. Launch EC2 instance with IAM role (S3 Full Access + SSM)
2. Configure security group for internal SSH access only (no public IP)
3. Create SSH key pair for on-prem connections
4. Connect via AWS Systems Manager (SSM) — no keys, no public IP needed
5. Test S3 access via CloudShell

### Part 2: S3 Bucket and Access Control (30 minutes)

6. Create S3 bucket
7. Look up your SSO role ARN (your existing identity — no new users needed)
8. Apply a bucket policy that allows your SSO role but denies the EC2 role
9. Demonstrate access denial from EC2 role
10. Test access from your laptop using your SSO session
11. Update bucket policy to also allow the EC2 role
12. Verify EC2 role access now works

---

## Part 1: EC2 Instance Setup (30 minutes)

### Step 1: Create IAM Role for EC2 (5 minutes)

**Create the IAM Role:**

1. Navigate to **IAM Console → Roles → Create role**
2. Select **AWS service → EC2 → Next**
3. Attach the following managed policies:
   - `AmazonSSMManagedInstanceCore` (for SSM access)
   - `AmazonS3FullAccess` (for full S3 access)
4. Name the role: `Workshop-EC2-S3-SSM-Role`
5. Click **Create role**
6. Copy the **Role ARN** — you'll need this later
   - Format: `arn:aws:iam::123456789012:role/Workshop-EC2-S3-SSM-Role`

> **What this does:**
> - SSM policy allows connection without SSH keys
> - S3 Full Access allows all S3 operations (read, write, delete)
> - No credentials stored on the instance — the role provides temporary credentials automatically

---

### Step 2: Create SSH Key Pair (3 minutes)

1. Navigate to **EC2 Console → Key Pairs → Create key pair**
2. Name: `workshop-keypair`
3. Key pair type: **RSA**
4. Private key file format:
   - `.pem` (for Mac/Linux/Windows with OpenSSH)
   - `.ppk` (for Windows with PuTTY)
5. Click **Create key pair**
6. Save the downloaded file securely — you can't download it again

For Mac/Linux users:

```bash
chmod 400 ~/Downloads/workshop-keypair.pem
```

---

### Step 3: Create Security Group (3 minutes)

1. Navigate to **EC2 Console → Security Groups → Create security group**
2. Name: `Workshop-Internal-Access-SG`
3. Description: `Allow SSH from internal network only`
4. VPC: Select your VPC

**Inbound rules:**

| Rule | Type | Port | Source | Description |
|------|------|------|--------|-------------|
| 1 | SSH | 22 | `YOUR-INTERNAL-CIDR-1` | SSH from internal network 1 |
| 2 | SSH | 22 | `YOUR-INTERNAL-CIDR-2` | SSH from internal network 2 |

**Outbound rules:** Leave default (all traffic allowed)

Click **Create security group**

> **What this does:**
> - Allows SSH only from your internal network ranges — no public internet exposure
> - SSM connections don't require any inbound rules, so no additional ports needed
> - Port 22 can be removed entirely if your team uses SSM exclusively

---

### Step 4: Launch EC2 Instance (5 minutes)

1. Navigate to **EC2 Console → Launch Instance**
2. Name: `workshop-data-eng-demo`
3. AMI: **Amazon Linux 2023** (free tier eligible)
4. Instance type: `t3.micro` or `t2.micro`
5. Key pair: Select `workshop-keypair`
6. Network settings → Click **Edit**:
   - Select your VPC
   - Select a private subnet
   - Auto-assign public IP: **Disable**
   - Firewall: Select existing security group → `Workshop-Internal-Access-SG`
7. Advanced details:
   - Scroll down to **IAM instance profile**
   - Select: `Workshop-EC2-S3-SSM-Role`
8. Click **Launch instance**

Wait 2–3 minutes for the instance to reach "Running" state and pass status checks.

> **No public IP — how does SSM work?**
> SSM Session Manager requires the instance to reach SSM service endpoints. Without a public IP this works via either:
> - **NAT Gateway** in your VPC (outbound internet through NAT)
> - **VPC Endpoints** for SSM (recommended for production — keeps all traffic private)
>
> Check with your network team which is in place. If neither exists, SSM won't connect and SSH from on-prem will be the only option.

---

### Step 5: Connect via SSH from On-Prem (5 minutes)

Since there's no public IP, SSH works only from your internal network directly to the instance's **private IP**.

**Get the private IP:**

1. Navigate to **EC2 Console → Instances**
2. Select your instance: `workshop-data-eng-demo`
3. Copy the **Private IPv4 address** (e.g., `10.x.x.x`)

**Connect:**

```bash
# Mac/Linux
ssh -i ~/Downloads/workshop-keypair.pem ec2-user@PRIVATE-IP

# Windows PowerShell (with OpenSSH)
ssh -i C:\Users\YourName\Downloads\workshop-keypair.pem ec2-user@PRIVATE-IP
```

**Windows (PuTTY):**

1. Open PuTTY
2. Host Name: `ec2-user@PRIVATE-IP`
3. Connection → SSH → Auth → Browse for your `.ppk` file
4. Click **Open**

If prompted about host authenticity, type `yes` and press Enter.

> **Must be on the internal network** — this only works if you're connected via VPN or on-prem. If you're not on the internal network, use SSM (Step 6) instead.

---

### Step 6: Connect via Systems Manager — Preferred Method (3 minutes)

SSM Session Manager is the recommended connection method — no public IP, no open ports, works from anywhere with AWS console access.

1. Navigate to **EC2 Console → Instances**
2. Select your instance: `workshop-data-eng-demo`
3. Click **Connect** button (top right)
4. Select **Session Manager** tab
5. Click **Connect**

You should see a terminal session open in your browser.

> **Why SSM is preferred over SSH here:**
>
> | | SSH | SSM |
> |---|---|---|
> | Requires public IP | No (internal only) | No |
> | Keys required | Yes | No |
> | Open inbound ports | Port 22 | None |
> | Audit trail | Manual | CloudTrail |
> | Works without VPN | No | Yes (via console) |
>
> If SSM fails to connect, verify the instance has outbound access to SSM endpoints (via NAT Gateway or VPC Endpoints) and that the `Workshop-EC2-S3-SSM-Role` is attached.

---

### Step 7: Install AWS CLI and Test S3 Access (6 minutes)

In your terminal (SSH or SSM), run these commands:

```bash
# Verify AWS CLI is installed (comes with Amazon Linux 2023)
aws --version

# Check the instance's IAM role
aws sts get-caller-identity

# List all S3 buckets (should work with our IAM role)
aws s3 ls

# Create a test file
echo "Hello from EC2 - $(date)" > test-file.txt
cat test-file.txt
```

**Expected output:**

- AWS CLI version displayed (e.g., `aws-cli/2.x.x`)
- IAM role ARN shown (should contain `Workshop-EC2-S3-SSM-Role`)
- List of S3 buckets in your account (if any exist)
- Test file created successfully

> **Key Takeaway:** The EC2 instance can access S3 using the IAM role — no credentials needed.

---

## Part 2: S3 Bucket and Access Control (30 minutes)

### Step 8: Look Up Your SSO Role ARN via CloudShell (3 minutes)

You already have an AWS identity through IAM Identity Center (SSO) — no new users or groups needed. We'll use **AWS CloudShell** to find your SSO role ARN. CloudShell runs directly in the browser, pre-authenticated as your logged-in identity, with AWS CLI already installed — no local setup required.

**Open CloudShell:**

- Click the CloudShell icon in the AWS Console top navigation bar (looks like `>_`)
- Or navigate to **CloudShell** from the Services menu
- Wait a few seconds for the shell to initialize

**Find your SSO role ARN:**

```bash
aws sts get-caller-identity
```

The output will look like this:

```json
{
  "UserId": "AROA...:your.name@example.com",
  "Account": "123456789012",
  "Arn": "arn:aws:sts::123456789012:assumed-role/AWSReservedSSO_YourPermissionSet_xxxx/your.name@example.com"
}
```

**Copy the base role ARN** — strip the session suffix (everything after the role name):

```
arn:aws:iam::123456789012:role/aws-reserved/sso.amazonaws.com/AWSReservedSSO_YourPermissionSet_xxxx
```

> **Tip:** You can also find this in **IAM Console → Roles → search for `AWSReservedSSO`**

> **What this shows:** Your SSO session is backed by an IAM role AWS assumes on your behalf. This is the principal we'll use in the bucket policy — no long-lived credentials, no IAM users needed.

---

### Step 9: Create S3 Bucket (3 minutes)

1. Navigate to **S3 Console → Create bucket**
2. Bucket name: `workshop-demo-bucket-[your-initials]-[random-number]`
   - Example: `workshop-demo-bucket-jd-12345`
   - Must be globally unique
3. Region: Select your region (same as EC2)
4. Object Ownership: **ACLs disabled** (recommended)
5. Block Public Access: **Keep all settings enabled** (default)
6. Bucket Versioning: **Disabled** (for this demo)
7. Encryption: **SSE-S3** (Amazon S3 managed keys)
8. Click **Create bucket**

---

### Step 10: Create Bucket Policy — Allow SSO Role, Deny EC2 Role (7 minutes)

We'll create a bucket policy that explicitly allows your SSO role but leaves the EC2 role out — demonstrating that the EC2 role's IAM-level `AmazonS3FullAccess` is overridden by the bucket policy Deny.

1. Go to **S3 Console → Select your bucket**
2. Click **Permissions** tab
3. Scroll to **Bucket policy → Click Edit**
4. Paste this policy (replace the placeholder values):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSSORole",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::YOUR-ACCOUNT-ID:role/aws-reserved/sso.amazonaws.com/YOUR-SSO-ROLE-NAME"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::YOUR-BUCKET-NAME",
        "arn:aws:s3:::YOUR-BUCKET-NAME/*"
      ]
    }
  ]
}
```

**Replace these values:**

| Placeholder | Value |
|-------------|-------|
| `YOUR-ACCOUNT-ID` | Your 12-digit AWS account ID |
| `YOUR-SSO-ROLE-NAME` | Your SSO permission set role name from Step 8 (e.g., `AWSReservedSSO_YourPermissionSet_xxxx`) |
| `YOUR-BUCKET-NAME` | Your actual bucket name |

Click **Save changes**

> **What this does:**
> - Explicitly allows your SSO role to perform S3 operations
> - S3 denies everyone else by default — no explicit Deny statement needed
> - The EC2 role has `AmazonS3FullAccess` via IAM, but the bucket policy doesn't allow it, so access is denied
> - This demonstrates that both IAM policy AND bucket policy must allow access for an operation to succeed

> **Scaling to a whole team:** To grant access to everyone in an SSO group, use a wildcard on the permission set name in a `Condition` block instead:
> ```json
> "Principal": "*",
> "Condition": { "ArnLike": { "aws:PrincipalArn": "arn:aws:iam::YOUR-ACCOUNT-ID:role/aws-reserved/sso.amazonaws.com/AWSReservedSSO_YourPermissionSet_*" } }
> ```
> Run `aws iam list-roles --query "Roles[?contains(RoleName, 'AWSReservedSSO')].RoleName"` to find the exact permission set name in your account.

---

### Step 11: Test Access from EC2 — Should Fail (5 minutes)

From your EC2 terminal (SSH or SSM):

```bash
# Set your bucket name as a variable
BUCKET_NAME="workshop-demo-bucket-jd-12345"

# Try to list bucket contents — THIS SHOULD FAIL
aws s3 ls s3://$BUCKET_NAME/

# Try to upload the test file — THIS SHOULD ALSO FAIL
aws s3 cp test-file.txt s3://$BUCKET_NAME/
```

**Expected result:** ❌ Access Denied errors

> **Why?**
> - The EC2 instance has `AmazonS3FullAccess` via its IAM role
> - But the bucket policy has an explicit **Deny** for all principals except the workshop user
> - **Explicit Deny always wins** — this is the most important IAM concept to understand

---

### Step 12: Test Access from CloudShell as Your SSO Identity — Should Work (5 minutes)

Now test access using CloudShell, which is already authenticated as your SSO identity — no profile or credentials needed.

**Open CloudShell** (if not already open) from the AWS Console top navigation bar.

```bash
# Confirm your identity — should show your SSO role
aws sts get-caller-identity

# Set your bucket name
BUCKET_NAME="workshop-demo-bucket-jd-12345"

# Test listing the bucket
aws s3 ls s3://$BUCKET_NAME/

# Upload a file
echo "Access from SSO identity via CloudShell" > sso-test.txt
aws s3 cp sso-test.txt s3://$BUCKET_NAME/

# Verify upload
aws s3 ls s3://$BUCKET_NAME/
```

**Expected result:** ✅ Access works — you can list and upload files.

> **Why?**
> - CloudShell uses your console login identity — the same SSO role ARN you put in the bucket policy
> - No local CLI, no credentials, no profile setup needed
> - This is your identity as a team member — completely independent from the EC2 role

---

### Step 13: Add EC2 Role to Bucket Policy (5 minutes)

Now let's grant access to the EC2 instance role by updating the bucket policy:

1. Go to **S3 Console → Your bucket → Permissions → Bucket policy → Edit**
2. Update the policy to allow **both** the user and the EC2 role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSSOAndEC2Role",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::YOUR-ACCOUNT-ID:role/aws-reserved/sso.amazonaws.com/YOUR-SSO-ROLE-NAME",
          "arn:aws:iam::YOUR-ACCOUNT-ID:role/Workshop-EC2-S3-SSM-Role"
        ]
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::YOUR-BUCKET-NAME",
        "arn:aws:s3:::YOUR-BUCKET-NAME/*"
      ]
    }
  ]
}
```

Click **Save changes**

---

### Step 14: Verify EC2 Access Now Works (5 minutes)

From your EC2 terminal, test again:

```bash
# This should NOW work
aws s3 ls s3://$BUCKET_NAME/

# Upload a new file
echo "Access from EC2 after policy update - $(date)" > ec2-success.txt
aws s3 cp ec2-success.txt s3://$BUCKET_NAME/

# List all files
aws s3 ls s3://$BUCKET_NAME/

# Download and verify
aws s3 cp s3://$BUCKET_NAME/ec2-success.txt downloaded.txt
cat downloaded.txt
```

**Expected result:** ✅ All commands work successfully

> **What changed?**
> - We added the EC2 role ARN to both the Allow statement and the Deny exception list
> - Now both the IAM user and the EC2 role can access the bucket
> - Everyone else is still denied

---

## Workshop Summary

### What We Learned

**EC2 Best Practices:**

- ✅ Use IAM roles instead of storing credentials on instances
- ✅ Support both SSH (traditional) and SSM (modern) connection methods
- ✅ Restrict security groups to internal networks only
- ✅ Attach roles at launch time for immediate access

**S3 Access Control Layers:**

- ✅ Layer 1 — IAM Policy: Defines what an identity *can* do (e.g., `AmazonS3FullAccess`)
- ✅ Layer 2 — Bucket Policy: Defines who *may* access the bucket (Allow + Deny)
- ✅ Explicit Deny always wins, even over IAM Allow

**Access Control Flow:**

| Scenario | Result | Why |
|----------|--------|-----|
| EC2 role alone (before policy update) | ❌ Denied | Bucket policy has no Allow for the EC2 role — default deny applies |
| SSO session from laptop | ✅ Granted | Bucket policy explicitly allows the SSO role |
| EC2 role after policy update | ✅ Granted | EC2 role added to the Allow statement |

**Security Layers:**

| Layer | Controls |
|-------|----------|
| Network | Security groups restrict IP access |
| Identity | IAM roles/policies define capabilities |
| Resource | Bucket policies control access to specific resources |

### Key Takeaways

**IAM Policy vs Bucket Policy:**

- **IAM Policy:** "What can this identity do?" (attached to roles, users, groups)
- **Bucket Policy:** "Who can access this bucket?" (attached to the S3 bucket)
- Both are evaluated — and **explicit Deny always wins**

**Access Decision Flow:**

1. Does the IAM role/user have permission for the action? (e.g., `s3:PutObject`)
2. Does the bucket policy allow this principal?
3. Is there an explicit Deny anywhere? → **Deny always wins**
4. If all checks pass → ✅ Access Granted

---

## Bonus: IAM Roles Anywhere — Extend AWS Access to On-Premises

**Did you know?** You can give your on-premises servers, dev boxes, and non-AWS workloads temporary AWS credentials — just like EC2 instances get from IAM roles.

### What is IAM Roles Anywhere?

IAM Roles Anywhere lets you use **X.509 certificates** issued by your own Certificate Authority (CA) to obtain temporary AWS credentials for workloads running outside of AWS.

### How It Works

1. **Register your CA** as a trust anchor in IAM Roles Anywhere
2. **Create a profile** that maps to an IAM role
3. **Install a credential helper** on your on-premises machine
4. Your workload presents its certificate → AWS validates it → temporary credentials issued

### Why This Matters for Data Engineering

- Run ETL scripts on on-premises servers that write directly to S3
- Access AWS services from dev laptops without long-lived access keys
- Same IAM role-based access model you use in AWS — extended everywhere
- Credentials are **temporary and automatically rotated**
- No VPN or Direct Connect required for credential issuance

### Quick Example

```bash
# Install the credential helper
# (available for Linux, macOS, Windows)

# Configure AWS CLI to use Roles Anywhere
aws configure set credential_process "/path/to/aws_signing_helper credential-process \
  --certificate /path/to/cert.pem \
  --private-key /path/to/key.pem \
  --trust-anchor-arn arn:aws:rolesanywhere:region:account:trust-anchor/ta-id \
  --profile-arn arn:aws:rolesanywhere:region:account:profile/profile-id \
  --role-arn arn:aws:iam::account:role/role-name"

# Now use AWS CLI normally — credentials are fetched automatically
aws s3 ls
```

> **Key benefit:** Eliminates long-lived access keys for any system that needs AWS access. This is the recommended approach for on-premises workloads, CI/CD pipelines, and developer machines.

---

## Cleanup (5 minutes)

To avoid charges, clean up resources:

| Resource | How to Delete |
|----------|---------------|
| EC2 instance | EC2 Console → Select instance → Instance State → Terminate |
| S3 bucket | S3 Console → Select bucket → Empty → then Delete |
| IAM role | IAM Console → Roles → `Workshop-EC2-S3-SSM-Role` → Delete |
| Security group | EC2 Console → Security Groups → `Workshop-Internal-Access-SG` → Delete |
| Key pair | EC2 Console → Key Pairs → `workshop-keypair` → Delete |

---

## Next Steps

- Explore **S3 encryption** with SSE-KMS for key management
- Learn about **S3 lifecycle policies** for automatic data archival
- Set up **CloudWatch alarms** for monitoring EC2 and S3
- Explore **VPC endpoints** for private S3 access (no internet gateway needed)
- Try **IAM Roles Anywhere** with a test certificate for on-premises access
- Practice with **AWS CLI** for automation and scripting

### Additional Resources

- [Amazon S3 User Guide](https://docs.aws.amazon.com/s3/)
- [Amazon EC2 User Guide](https://docs.aws.amazon.com/ec2/)
- [IAM Roles Anywhere User Guide](https://docs.aws.amazon.com/rolesanywhere/)
- [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [AWS Skill Builder](https://skillbuilder.aws) (free training)
- [AWS Workshops](https://workshops.aws)

---

## Q&A

**Q: Why use SSM instead of SSH?**
A: No key management, no open ports, all sessions logged in CloudTrail, works from anywhere with console access.

**Q: Do we need a public IP for SSM?**
A: Not if you set up VPC endpoints (AWS PrivateLink). With VPC endpoints, all SSM traffic stays private. We used a public IP here for simplicity.

**Q: Why can't we use IAM groups in bucket policies?**
A: S3 bucket policies only support users, roles, accounts, and services as principals — not groups. Using your SSO role ARN directly is the right approach, and it's actually more precise since it maps to a specific identity.

**Q: What if we need to allow access from a Lambda function?**
A: Add the Lambda execution role ARN to the bucket policy's Principal list and the Deny exception.

**Q: How do we audit who accessed what in S3?**
A: Enable CloudTrail (logs all API calls with identity) and S3 Server Access Logging (logs all requests to the bucket).

**Q: Can we restrict access by IP address in a bucket policy?**
A: Yes — use the `aws:SourceIp` condition key to allow/deny based on IP ranges.

**Q: How does IAM Roles Anywhere help our on-premises systems?**
A: It lets any system with an X.509 certificate from your CA obtain temporary AWS credentials. No long-lived access keys, no VPN required. Perfect for ETL servers, dev boxes, and CI/CD pipelines.
