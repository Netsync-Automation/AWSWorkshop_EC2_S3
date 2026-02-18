# AWS Workshop Quick Reference Card

**EC2 and S3 Fundamentals — Command Cheat Sheet**

---

## EC2 Connection Commands

### SSH Connection

```bash
# Mac/Linux — connect to private IP (must be on internal network or VPN)
chmod 400 /path/to/workshop-keypair.pem
ssh -i /path/to/workshop-keypair.pem ec2-user@PRIVATE-IP

# Windows PowerShell (with OpenSSH)
ssh -i C:\path\to\workshop-keypair.pem ec2-user@PRIVATE-IP
```

### SSM Connection

1. EC2 Console → Select instance → **Connect** button
2. Choose **Session Manager** tab → **Connect**
3. Browser-based terminal opens automatically

---

## AWS CLI — S3 Commands

### Basic Operations

```bash
BUCKET_NAME="workshop-demo-bucket-jd-12345"

aws s3 ls                                          # List all buckets
aws s3 ls s3://$BUCKET_NAME/                       # List bucket contents
aws s3 cp local-file.txt s3://$BUCKET_NAME/        # Upload file
aws s3 cp s3://$BUCKET_NAME/file.txt ./            # Download file
aws s3 rm s3://$BUCKET_NAME/file.txt               # Delete file
aws s3 sync ./local-folder s3://$BUCKET_NAME/      # Sync directory up
aws s3 sync s3://$BUCKET_NAME/folder/ ./local/     # Sync directory down
```

### Advanced Operations

```bash
aws s3 cp s3://source-bucket/f.txt s3://dest-bucket/   # Copy between buckets
aws s3 mv s3://$BUCKET_NAME/old.txt s3://$BUCKET_NAME/new.txt  # Move/rename
aws s3 ls s3://$BUCKET_NAME/ --human-readable          # Human-readable sizes
aws s3 rm s3://$BUCKET_NAME/folder/ --recursive         # Delete folder
```

---

## AWS CLI — IAM and Identity Commands

```bash
# Check current identity
aws sts get-caller-identity

# CloudShell — open from AWS Console top nav bar (>_ icon)
# No setup needed — already authenticated as your logged-in identity
aws sts get-caller-identity       # Confirm your SSO identity
aws s3 ls s3://bucket-name/       # No profile flags needed

# IAM info
aws iam list-roles
aws iam list-groups
aws iam list-users
aws iam get-role --role-name Workshop-EC2-S3-SSM-Role
```

---

## AWS CLI — EC2 Commands

```bash
# List instances (table format)
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# Instance lifecycle
aws ec2 start-instances --instance-ids i-1234567890abcdef0
aws ec2 stop-instances --instance-ids i-1234567890abcdef0
aws ec2 reboot-instances --instance-ids i-1234567890abcdef0
aws ec2 terminate-instances --instance-ids i-1234567890abcdef0

# Get instance public IP
aws ec2 describe-instances \
  --instance-ids i-1234567890abcdef0 \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text
```

---

## Security Group CIDR Blocks

| Network | CIDR |
|---------|------|
| Internal Network 1 | `YOUR-INTERNAL-CIDR-1` |
| Internal Network 2 | `YOUR-INTERNAL-CIDR-2` |

**Common Private IP Ranges:**

| Class | Range |
|-------|-------|
| A | `10.0.0.0/8` (10.0.0.0 – 10.255.255.255) |
| B | `172.16.0.0/12` (172.16.0.0 – 172.31.255.255) |
| C | `192.168.0.0/16` (192.168.0.0 – 192.168.255.255) |

---

## IAM Policy Examples

### S3 Bucket-Specific Access

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket", "s3:DeleteObject"],
      "Resource": [
        "arn:aws:s3:::bucket-name",
        "arn:aws:s3:::bucket-name/*"
      ]
    }
  ]
}
```

### Bucket Policy — SSO Role and EC2 Role with Deny

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSpecificPrincipals",
      "Effect": "Allow",
      "Principal": "*",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket", "s3:DeleteObject"],
      "Resource": ["arn:aws:s3:::bucket-name", "arn:aws:s3:::bucket-name/*"],
      "Condition": {
        "ArnLike": {
          "aws:PrincipalArn": [
            "arn:aws:iam::123456789012:role/aws-reserved/sso.amazonaws.com/AWSReservedSSO_YourPermissionSet_xxxx",
            "arn:aws:iam::123456789012:role/Workshop-EC2-S3-SSM-Role"
          ]
        }
      }
    },
    {
      "Sid": "DenyAllOthers",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": ["arn:aws:s3:::bucket-name", "arn:aws:s3:::bucket-name/*"],
      "Condition": {
        "ArnNotLike": {
          "aws:PrincipalArn": [
            "arn:aws:iam::123456789012:role/aws-reserved/sso.amazonaws.com/AWSReservedSSO_YourPermissionSet_xxxx",
            "arn:aws:iam::123456789012:role/Workshop-EC2-S3-SSM-Role",
            "arn:aws:iam::123456789012:root"
          ]
        }
      }
    }
  ]
}
```

---

## ARN Formats

| Resource | Format |
|----------|--------|
| IAM Role | `arn:aws:iam::ACCOUNT-ID:role/RoleName` |
| IAM User | `arn:aws:iam::ACCOUNT-ID:user/UserName` |
| IAM Group | `arn:aws:iam::ACCOUNT-ID:group/GroupName` |
| IAM Policy | `arn:aws:iam::ACCOUNT-ID:policy/PolicyName` |
| S3 Bucket | `arn:aws:s3:::bucket-name` |
| S3 Objects | `arn:aws:s3:::bucket-name/*` |
| EC2 Instance | `arn:aws:ec2:region:ACCOUNT-ID:instance/i-xxx` |

---

## Workshop Resource Names

| Resource | Name |
|----------|------|
| IAM Role | `Workshop-EC2-S3-SSM-Role` |
| EC2 Instance | `workshop-data-eng-demo` |
| Key Pair | `workshop-keypair` |
| Security Group | `Workshop-Internal-Access-SG` |
| S3 Bucket | `workshop-demo-bucket-[initials]-[number]` |
| SSO Role | Look up with `aws sts get-caller-identity` |

---

## Access Decision Flow

For any AWS API call, access is granted only if:

1. ✅ IAM policy allows the action (e.g., `s3:GetObject`)
2. ✅ Bucket policy allows the principal (e.g., specific role ARN)
3. ✅ No explicit Deny exists anywhere
4. ✅ All conditions are met (e.g., IP restrictions, MFA)

> ⚠️ **Explicit Deny always wins over Allow**

---

## Useful Linux Commands (on EC2)

```bash
echo "Hello World" > test.txt     # Create test file
cat test.txt                       # View file
ls -lah                            # List files
df -h                              # Disk space
pwd                                # Current directory
cat /etc/os-release                # OS version
free -h                            # Memory usage
```

---

## Emergency Cleanup

```bash
aws ec2 terminate-instances --instance-ids i-XXXXX
aws s3 rm s3://bucket-name/ --recursive
aws s3 rb s3://bucket-name
```

---

## Additional Resources

- [S3 User Guide](https://docs.aws.amazon.com/s3/)
- [EC2 User Guide](https://docs.aws.amazon.com/ec2/)
- [IAM Roles Anywhere](https://docs.aws.amazon.com/rolesanywhere/)
- [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [AWS Skill Builder](https://skillbuilder.aws) (free training)
