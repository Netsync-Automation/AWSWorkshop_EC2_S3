# AWS Workshop Troubleshooting Guide

**Common Issues and Solutions**

---

## EC2 Connection Issues

### Cannot connect via SSH — "Connection timed out"

**Possible causes:** Security group doesn't allow SSH from your IP, instance not running, wrong public IP, network issues.

**Solutions:**

```bash
# Check security group rules
aws ec2 describe-security-groups \
  --group-ids sg-XXXXX \
  --query 'SecurityGroups[0].IpPermissions'

# Check if instance is running
aws ec2 describe-instances \
  --instance-ids i-XXXXX \
  --query 'Reservations[0].Instances[0].State.Name'

# Get current public IP (changes on stop/start)
aws ec2 describe-instances \
  --instance-ids i-XXXXX \
  --query 'Reservations[0].Instances[0].PrivateIpAddress'
```

- Ensure port 22 is open for `YOUR-INTERNAL-CIDR-1` or `YOUR-INTERNAL-CIDR-2`
- Confirm you are on the internal network or connected via VPN
- If not on the internal network, use SSM Session Manager instead

---

### SSH "Permission denied (publickey)"

**Possible causes:** Wrong key file, incorrect permissions, wrong username.

```bash
# Fix key permissions (Mac/Linux)
chmod 400 /path/to/workshop-keypair.pem

# Correct command
ssh -i /path/to/workshop-keypair.pem ec2-user@PUBLIC-IP
```

**Default usernames by AMI:**

| AMI | Username |
|-----|----------|
| Amazon Linux 2023 | `ec2-user` |
| Ubuntu | `ubuntu` |
| Red Hat | `ec2-user` |
| SUSE | `ec2-user` |

Verify the key pair name in EC2 Console → Instance details → Key pair name.

---

### Cannot connect via SSM Session Manager

**Possible causes:** IAM role missing, SSM Agent not running, no internet connectivity.

```bash
# Check IAM role is attached
aws ec2 describe-instances \
  --instance-ids i-XXXXX \
  --query 'Reservations[0].Instances[0].IamInstanceProfile'

# Check SSM Agent status (via SSH)
sudo systemctl status amazon-ssm-agent

# Restart if needed
sudo systemctl start amazon-ssm-agent

# Test SSM endpoint connectivity
curl -I https://ssm.us-east-1.amazonaws.com
```

- Should show `Workshop-EC2-S3-SSM-Role`
- If missing: EC2 Console → Actions → Security → Modify IAM role
- Check **Systems Manager → Fleet Manager** for your instance
- If not listed, wait 5–10 minutes or restart SSM Agent

---

## S3 Access Issues

### "Access Denied" when listing or accessing bucket

**Possible causes:** Bucket policy Deny blocking access, IAM role not in exception list, wrong credentials.

```bash
# Verify current identity
aws sts get-caller-identity

# Test with specific bucket
aws s3 ls s3://workshop-demo-bucket-jd-12345/
```

**Check these in order:**

1. **Bucket policy** — S3 Console → Bucket → Permissions → Bucket policy
   - Is your SSO role ARN in the Allow statement?
   - Is your SSO role ARN in the Deny exception (`StringNotLike`)?
   - SSO role ARN format: `arn:aws:iam::ACCOUNT:role/aws-reserved/sso.amazonaws.com/AWSReservedSSO_xxx`
2. **Correct SSO profile** — Are you using the right `--profile` flag?
3. **SSO session active** — Run `aws sso login --profile your-profile` if expired

**Required S3 actions by operation:**

| Operation | Required Action | Resource |
|-----------|----------------|----------|
| List bucket | `s3:ListBucket` | Bucket ARN |
| Get object | `s3:GetObject` | Bucket ARN/* |
| Put object | `s3:PutObject` | Bucket ARN/* |
| Delete object | `s3:DeleteObject` | Bucket ARN/* |

---

### Bucket policy changes not taking effect

**Possible causes:** JSON syntax error, policy not saved, caching delay.

**Common JSON mistakes:**

```json
// ❌ WRONG — missing comma
{
  "Effect": "Allow"
  "Action": "s3:GetObject"
}

// ✅ CORRECT
{
  "Effect": "Allow",
  "Action": "s3:GetObject"
}
```

- Use an online JSON validator
- Check for missing commas, brackets, quotes
- Verify all ARNs are correct format
- Wait 1–2 minutes for propagation
- Confirm changes are saved in the console

---

## AWS CLI Issues

### "aws: command not found"

```bash
# Verify installation
which aws
aws --version

# Install on Amazon Linux 2023 (should be pre-installed)
sudo yum install aws-cli -y
```

---

### "Unable to locate credentials"

```bash
# On EC2 — should use IAM role
aws sts get-caller-identity

# If using profiles
aws configure list-profiles
aws s3 ls --profile profile-name
```

- EC2 instances should use IAM roles, not stored credentials
- Verify role is attached: EC2 Console → Instance → Actions → Security → Modify IAM role

---

### "InvalidAccessKeyId"

```bash
# On EC2 — remove any stored credentials (use IAM role instead)
rm -rf ~/.aws/credentials
aws sts get-caller-identity

# For workshop user profile — re-configure
aws configure --profile workshop-user
```

---

## IAM Role and Policy Issues

### IAM role not working after attaching

- Wait 1–2 minutes for propagation
- Verify from instance:

```bash
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

- Check attached policies:

```bash
aws iam list-attached-role-policies --role-name Workshop-EC2-S3-SSM-Role
```

---

### "User is not authorized to perform: iam:PassRole"

- Use AWS Console to attach the role (easier than CLI)
- Or ask administrator to grant PassRole permission

---

## Network and Connectivity Issues

### Cannot reach AWS services from EC2

```bash
# Test internet
ping -c 4 8.8.8.8

# Test AWS endpoint
curl -I https://s3.amazonaws.com
```

**Check:**

- Internet gateway attached to VPC
- Route table has `0.0.0.0/0 → igw-XXXXX`
- Security group outbound rules allow all traffic
- Network ACLs aren't blocking

---

## Cleanup Issues

### Cannot delete S3 bucket — "BucketNotEmpty"

```bash
# Empty bucket first
aws s3 rm s3://bucket-name/ --recursive

# Then delete
aws s3 rb s3://bucket-name
```

### Cannot delete security group — "DependencyViolation"

```bash
# Find instances using this security group
aws ec2 describe-instances \
  --filters "Name=instance.group-id,Values=sg-XXXXX" \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name]'
```

Terminate or modify instances first, then delete the security group.

---

## Diagnostic Commands Summary

```bash
# Identity and permissions
aws sts get-caller-identity
aws iam get-role --role-name Workshop-EC2-S3-SSM-Role

# EC2 status
aws ec2 describe-instances --instance-ids i-XXXXX
aws ec2 describe-instance-status --instance-ids i-XXXXX

# S3 access
aws s3 ls
aws s3 ls s3://bucket-name/

# Network
ping -c 4 8.8.8.8
curl -I https://s3.amazonaws.com

# System (from EC2)
sudo systemctl status amazon-ssm-agent
df -h
free -h
```

---

## Prevention Best Practices

**Before starting:**

- ✅ Verify IAM role has required policies
- ✅ Confirm security group rules are correct
- ✅ Check instance is in correct VPC/subnet
- ✅ Ensure route table has internet gateway route

**During workshop:**

- ✅ Test each step before moving to next
- ✅ Keep AWS Console open for verification
- ✅ Save all ARNs and IDs in a text file
- ✅ Use consistent naming conventions

**After workshop:**

- ✅ Stop instances when not in use
- ✅ Delete test data from S3
- ✅ Review CloudTrail logs for errors

---

## Getting Help

| Resource | Link |
|----------|------|
| S3 Documentation | https://docs.aws.amazon.com/s3/ |
| EC2 Documentation | https://docs.aws.amazon.com/ec2/ |
| IAM Documentation | https://docs.aws.amazon.com/iam/ |
| IAM Roles Anywhere | https://docs.aws.amazon.com/rolesanywhere/ |
| Systems Manager | https://docs.aws.amazon.com/systems-manager/ |
| AWS Skill Builder | https://skillbuilder.aws |
| AWS re:Post | https://repost.aws |

---

## Emergency Contacts

- Workshop Instructor: [Contact information]
- AWS Support: [Your organization's support contact]
- AWS Status Dashboard: https://status.aws.amazon.com
- Security Issues: [Security team contact]
