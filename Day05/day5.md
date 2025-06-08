# Day 5: AWS Security - Complete Guide

## 📋 Table of Contents
- [Overview](#overview)
- [Learning Objectives](#learning-objectives)
- [AWS Security Fundamentals](#aws-security-fundamentals)
- [Identity and Access Management (IAM)](#identity-and-access-management-iam)
- [Network Security](#network-security)
- [Data Protection](#data-protection)
- [Monitoring and Compliance](#monitoring-and-compliance)
- [Hands-on Labs](#hands-on-labs)
- [Best Practices](#best-practices)
- [Common Security Pitfalls](#common-security-pitfalls)
- [Additional Resources](#additional-resources)

## 🎯 Overview

AWS Security is built on the **Shared Responsibility Model**, where AWS manages the security **of** the cloud, while customers are responsible for security **in** the cloud. This day focuses on implementing comprehensive security measures to protect your AWS infrastructure and applications.

## 📚 Learning Objectives

By the end of this day, you will be able to:
- Understand the AWS Shared Responsibility Model
- Configure and manage IAM users, groups, roles, and policies
- Implement network security using Security Groups and NACLs
- Set up data encryption at rest and in transit
- Monitor security events and maintain compliance
- Apply security best practices across AWS services

## 🔐 AWS Security Fundamentals

### Shared Responsibility Model

**AWS Responsibilities (Security OF the Cloud):**
- Physical security of data centers
- Hardware and software infrastructure
- Network infrastructure
- Hypervisor patching and management

**Customer Responsibilities (Security IN the Cloud):**
- Operating system patches and updates
- Application security
- Identity and access management
- Network and firewall configuration
- Data encryption

### Core Security Principles

1. **Principle of Least Privilege**: Grant minimum permissions necessary
2. **Defense in Depth**: Multiple layers of security controls
3. **Fail Securely**: Systems should fail to a secure state
4. **Security by Design**: Build security into architecture from the start

## 👤 Identity and Access Management (IAM)

### IAM Components

#### Users
Individual people or services that need access to AWS resources.

```json
{
  "Version": "2012-10-17", 
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

#### Groups
Collections of users with similar access requirements.

**Example Groups:**
- `Developers`: Development environment access
- `Administrators`: Full administrative access
- `ReadOnlyUsers`: Read-only access to resources

#### Roles
Temporary security credentials for services or cross-account access.

**Common Role Types:**
- **Service Roles**: For AWS services (EC2, Lambda, etc.)
- **Cross-Account Roles**: For access between AWS accounts
- **Identity Provider Roles**: For federated access

#### Policies
JSON documents that define permissions.

**Policy Types:**
- **AWS Managed Policies**: Pre-built by AWS
- **Customer Managed Policies**: Created and managed by you
- **Inline Policies**: Embedded directly in users, groups, or roles

### IAM Best Practices

1. **Enable MFA (Multi-Factor Authentication)**
   ```bash
   # Enable MFA for root account and all users
   aws iam enable-mfa-device --user-name username --serial-number arn:aws:iam::123456789012:mfa/username
   ```

2. **Use Strong Password Policies**
   - Minimum 14 characters
   - Require uppercase, lowercase, numbers, and symbols
   - Password expiration and history

3. **Regular Access Reviews**
   - Review and remove unused users and roles
   - Audit permissions regularly
   - Use AWS Access Analyzer

4. **Implement Resource-Based Policies**
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "AllowSpecificUser",
         "Effect": "Allow",
         "Principal": {
           "AWS": "arn:aws:iam::123456789012:user/specific-user"
         },
         "Action": "s3:GetObject",
         "Resource": "arn:aws:s3:::secure-bucket/*"
       }
     ]
   }
   ```

## 🛡️ Network Security

### Security Groups

Security Groups act as virtual firewalls for EC2 instances, controlling inbound and outbound traffic at the instance level.

#### Security Group Rules

**Inbound Rules Example:**
| Type | Protocol | Port Range | Source | Description |
|------|----------|------------|---------|-------------|
| HTTP | TCP | 80 | 0.0.0.0/0 | Allow web traffic |
| HTTPS | TCP | 443 | 0.0.0.0/0 | Allow secure web traffic |
| SSH | TCP | 22 | 10.0.0.0/8 | Allow SSH from private network |

**Outbound Rules Example:**
| Type | Protocol | Port Range | Destination | Description |
|------|----------|------------|-------------|-------------|
| All Traffic | All | All | 0.0.0.0/0 | Allow all outbound |

#### Security Group Best Practices

1. **Principle of Least Privilege**: Only allow necessary ports and protocols
2. **Use Specific Source/Destination**: Avoid 0.0.0.0/0 when possible
3. **Reference Other Security Groups**: Instead of IP ranges
4. **Regular Reviews**: Audit and clean up unused rules

```bash
# Create security group
aws ec2 create-security-group \
    --group-name web-servers \
    --description "Security group for web servers"

# Add inbound rule
aws ec2 authorize-security-group-ingress \
    --group-id sg-12345678 \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0
```

### Network ACLs (Access Control Lists)

NACLs provide subnet-level security, acting as an additional layer of defense.

#### NACL vs Security Groups

| Feature | Security Groups | NACLs |
|---------|----------------|-------|
| Level | Instance | Subnet |
| Rules | Allow only | Allow and Deny |
| State | Stateful | Stateless |
| Order | All rules evaluated | Rules processed in order |

#### NACL Configuration Example

```bash
# Create network ACL
aws ec2 create-network-acl --vpc-id vpc-12345678

# Add inbound rule
aws ec2 create-network-acl-entry \
    --network-acl-id acl-12345678 \
    --rule-number 100 \
    --protocol tcp \
    --port-range From=80,To=80 \
    --cidr-block 0.0.0.0/0 \
    --rule-action allow
```

### VPC Security Features

#### VPC Flow Logs
Monitor network traffic for security analysis and troubleshooting.

```bash
# Enable VPC Flow Logs
aws ec2 create-flow-logs \
    --resource-type VPC \
    --resource-ids vpc-12345678 \
    --traffic-type ALL \
    --log-destination-type cloud-watch-logs \
    --log-group-name VPCFlowLogs
```

#### AWS WAF (Web Application Firewall)
Protects web applications from common web exploits.

**WAF Rule Examples:**
- SQL injection protection
- Cross-site scripting (XSS) prevention
- Rate limiting
- IP reputation lists

## 🔒 Data Protection

### Encryption at Rest

#### S3 Bucket Encryption
```bash
# Enable default encryption
aws s3api put-bucket-encryption \
    --bucket my-secure-bucket \
    --server-side-encryption-configuration '{
        "Rules": [
            {
                "ApplyServerSideEncryptionByDefault": {
                    "SSEAlgorithm": "AES256"
                }
            }
        ]
    }'
```

#### EBS Volume Encryption
```bash
# Create encrypted EBS volume
aws ec2 create-volume \
    --size 10 \
    --volume-type gp3 \
    --availability-zone us-east-1a \
    --encrypted
```

#### RDS Encryption
```bash
# Create encrypted RDS instance
aws rds create-db-instance \
    --db-instance-identifier mydb \
    --db-instance-class db.t3.micro \
    --engine mysql \
    --storage-encrypted \
    --kms-key-id arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012
```

### Encryption in Transit

#### SSL/TLS Certificates
Use AWS Certificate Manager (ACM) for free SSL certificates.

```bash
# Request certificate
aws acm request-certificate \
    --domain-name example.com \
    --subject-alternative-names *.example.com \
    --validation-method DNS
```

#### Application Load Balancer HTTPS
```bash
# Create HTTPS listener
aws elbv2 create-listener \
    --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-load-balancer/50dc6c495c0c9188 \
    --protocol HTTPS \
    --port 443 \
    --certificates CertificateArn=arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012
```

### AWS Key Management Service (KMS)

#### Customer Managed Keys
```bash
# Create KMS key
aws kms create-key \
    --description "My application encryption key" \
    --key-usage ENCRYPT_DECRYPT

# Create alias
aws kms create-alias \
    --alias-name alias/my-app-key \
    --target-key-id 12345678-1234-1234-1234-123456789012
```

#### Key Rotation
```bash
# Enable automatic key rotation
aws kms enable-key-rotation \
    --key-id 12345678-1234-1234-1234-123456789012
```

## 📊 Monitoring and Compliance

### AWS CloudTrail

CloudTrail provides audit logs of API calls made in your AWS account.

```bash
# Create CloudTrail
aws cloudtrail create-trail \
    --name my-trail \
    --s3-bucket-name my-cloudtrail-bucket \
    --include-global-service-events \
    --is-multi-region-trail
```

### AWS Config

Monitors resource configurations and compliance.

```bash
# Put configuration recorder
aws configservice put-configuration-recorder \
    --configuration-recorder name=default,roleARN=arn:aws:iam::123456789012:role/config-role \
    --recording-group allSupported=true,includeGlobalResourceTypes=true
```

### AWS GuardDuty

Threat detection service using machine learning.

```bash
# Enable GuardDuty
aws guardduty create-detector \
    --enable \
    --finding-publishing-frequency FIFTEEN_MINUTES
```

### AWS Security Hub

Centralized security findings dashboard.

```bash
# Enable Security Hub
aws securityhub enable-security-hub \
    --enable-default-standards
```

## 🧪 Hands-on Labs

### Lab 1: IAM Setup and Configuration

1. **Create IAM User**
   ```bash
   aws iam create-user --user-name lab-user
   ```

2. **Create and Attach Policy**
   ```bash
   aws iam create-policy --policy-name S3ReadOnlyPolicy --policy-document file://s3-readonly-policy.json
   aws iam attach-user-policy --user-name lab-user --policy-arn arn:aws:iam::123456789012:policy/S3ReadOnlyPolicy
   ```

3. **Enable MFA**
   ```bash
   aws iam enable-mfa-device --user-name lab-user --serial-number arn:aws:iam::123456789012:mfa/lab-user
   ```

### Lab 2: Network Security Implementation

1. **Create Security Group**
   ```bash
   aws ec2 create-security-group --group-name web-sg --description "Web server security group"
   ```

2. **Configure Inbound Rules**
   ```bash
   aws ec2 authorize-security-group-ingress --group-id sg-12345678 --protocol tcp --port 80 --cidr 0.0.0.0/0
   aws ec2 authorize-security-group-ingress --group-id sg-12345678 --protocol tcp --port 443 --cidr 0.0.0.0/0
   ```

3. **Create and Configure NACL**
   ```bash
   aws ec2 create-network-acl --vpc-id vpc-12345678
   aws ec2 create-network-acl-entry --network-acl-id acl-12345678 --rule-number 100 --protocol tcp --port-range From=80,To=80 --cidr-block 0.0.0.0/0 --rule-action allow
   ```

### Lab 3: Data Encryption Setup

1. **Create KMS Key**
   ```bash
   aws kms create-key --description "Lab encryption key"
   aws kms create-alias --alias-name alias/lab-key --target-key-id 12345678-1234-1234-1234-123456789012
   ```

2. **Enable S3 Bucket Encryption**
   ```bash
   aws s3api put-bucket-encryption --bucket lab-bucket --server-side-encryption-configuration file://encryption-config.json
   ```

3. **Create Encrypted EBS Volume**
   ```bash
   aws ec2 create-volume --size 10 --volume-type gp3 --availability-zone us-east-1a --encrypted --kms-key-id alias/lab-key
   ```

## ✅ Best Practices

### Identity and Access Management
- Use IAM roles instead of long-term access keys
- Implement MFA for all users
- Regular access reviews and cleanup
- Use AWS SSO for centralized access management
- Implement cross-account roles for multi-account strategies

### Network Security
- Use security groups as primary firewall
- Implement NACLs for additional subnet-level protection
- Enable VPC Flow Logs for monitoring
- Use AWS WAF for web application protection
- Implement DDoS protection with AWS Shield

### Data Protection
- Enable encryption at rest for all data stores
- Use SSL/TLS for data in transit
- Implement proper key management with KMS
- Regular backup and recovery testing
- Use AWS Secrets Manager for credentials

### Monitoring and Incident Response
- Enable CloudTrail in all regions
- Set up CloudWatch alarms for security events
- Use AWS Config for compliance monitoring
- Implement automated incident response
- Regular security assessments and penetration testing

## ⚠️ Common Security Pitfalls

### 1. Overprivileged Access
**Problem**: Granting excessive permissions
**Solution**: Implement least privilege principle

### 2. Weak Password Policies
**Problem**: Default or weak password requirements
**Solution**: Enforce strong password policies and MFA

### 3. Unencrypted Data
**Problem**: Storing sensitive data without encryption
**Solution**: Enable encryption at rest and in transit

### 4. Public S3 Buckets
**Problem**: Accidentally making S3 buckets public
**Solution**: Use S3 Block Public Access and bucket policies

### 5. Unused Resources
**Problem**: Leaving unused resources running
**Solution**: Regular audits and automated cleanup

### 6. Missing Monitoring
**Problem**: No visibility into security events
**Solution**: Enable comprehensive logging and monitoring

### 7. Hardcoded Credentials
**Problem**: Embedding access keys in code
**Solution**: Use IAM roles and AWS Secrets Manager

## 📖 Additional Resources

### AWS Documentation
- [AWS Security Best Practices](https://docs.aws.amazon.com/security/)
- [IAM User Guide](https://docs.aws.amazon.com/IAM/latest/UserGuide/)
- [VPC Security Guide](https://docs.aws.amazon.com/vpc/latest/userguide/security.html)
- [AWS KMS Developer Guide](https://docs.aws.amazon.com/kms/latest/developerguide/)

### Training and Certification
- AWS Certified Security - Specialty
- AWS Security Fundamentals (Digital Training)
- AWS Well-Architected Security Pillar

### Tools and Services
- AWS Security Hub
- AWS GuardDuty
- AWS Inspector
- AWS Trusted Advisor
- AWS Config

### Community Resources
- AWS Security Blog
- AWS re:Invent Security Sessions
- AWS Security Whitepapers
- Open Source Security Tools

## 🎯 Summary

AWS Security is a comprehensive topic that requires understanding of multiple layers of protection. The key takeaways from Day 5 include:

- Security is a shared responsibility between AWS and customers
- IAM is the foundation of AWS security
- Network security requires multiple layers (Security Groups, NACLs, WAF)
- Data must be protected both at rest and in transit
- Continuous monitoring and compliance are essential
- Regular security assessments and updates are critical

Remember that security is not a one-time setup but an ongoing process that requires constant attention, updates, and improvements as your AWS environment evolves.

---

**Next Steps**: Practice implementing these security measures in your AWS environment, starting with the hands-on labs and gradually building more complex security architectures.