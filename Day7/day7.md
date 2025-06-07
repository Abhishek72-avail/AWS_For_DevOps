# Day 7: Secure VPC Setup with EC2 Instances

## Project Overview

This project focuses on designing and implementing a secure Virtual Private Cloud (VPC) architecture with EC2 instances. You'll learn to create a multi-tier network infrastructure with proper security controls, routing, and access management.

## Learning Objectives

By completing this project, you will:
- Understand VPC networking fundamentals
- Configure public and private subnets
- Implement network security using Security Groups and NACLs
- Set up internet connectivity and NAT gateways
- Configure IAM roles for EC2 instances
- Establish secure SSH access

## Architecture Overview

```
Internet Gateway
       |
   Public Subnet (10.0.1.0/24)
       |
   [Web Server EC2]
       |
   Private Subnet (10.0.2.0/24)
       |
   [Database Server EC2]
       |
   NAT Gateway (for outbound access)
```

## Prerequisites

- AWS Account with appropriate permissions
- AWS CLI configured
- Basic understanding of networking concepts
- SSH client installed

## Step-by-Step Implementation

### Phase 1: VPC Design and Configuration

#### 1.1 Create VPC
```bash
# Create VPC with custom CIDR block
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=MySecureVPC}]'
```

**Key Concepts:**
- **CIDR Block**: Defines the IP address range for your VPC
- **10.0.0.0/16**: Provides 65,536 IP addresses
- VPC acts as your isolated network environment in AWS

#### 1.2 Create Subnets

**Public Subnet (Web Tier):**
```bash
# Create public subnet
aws ec2 create-subnet --vpc-id vpc-xxxxxxxx --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PublicSubnet}]'
```

**Private Subnet (Database Tier):**
```bash
# Create private subnet
aws ec2 create-subnet --vpc-id vpc-xxxxxxxx --cidr-block 10.0.2.0/24 --availability-zone us-east-1b --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PrivateSubnet}]'
```

**Subnet Planning:**
- Public Subnet: Resources that need direct internet access
- Private Subnet: Backend resources (databases, application servers)
- Different AZs provide high availability

### Phase 2: Internet Connectivity Setup

#### 2.1 Internet Gateway Configuration
```bash
# Create Internet Gateway
aws ec2 create-internet-gateway --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=MyIGW}]'

# Attach to VPC
aws ec2 attach-internet-gateway --internet-gateway-id igw-xxxxxxxx --vpc-id vpc-xxxxxxxx
```

#### 2.2 NAT Gateway Setup
```bash
# Allocate Elastic IP for NAT Gateway
aws ec2 allocate-address --domain vpc

# Create NAT Gateway in public subnet
aws ec2 create-nat-gateway --subnet-id subnet-xxxxxxxx --allocation-id eipalloc-xxxxxxxx --tag-specifications 'ResourceType=nat-gateway,Tags=[{Key=Name,Value=MyNATGW}]'
```

### Phase 3: Route Table Configuration

#### 3.1 Public Route Table
```bash
# Create route table for public subnet
aws ec2 create-route-table --vpc-id vpc-xxxxxxxx --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=PublicRouteTable}]'

# Add route to Internet Gateway
aws ec2 create-route --route-table-id rtb-xxxxxxxx --destination-cidr-block 0.0.0.0/0 --gateway-id igw-xxxxxxxx

# Associate with public subnet
aws ec2 associate-route-table --subnet-id subnet-xxxxxxxx --route-table-id rtb-xxxxxxxx
```

#### 3.2 Private Route Table
```bash
# Create route table for private subnet
aws ec2 create-route-table --vpc-id vpc-xxxxxxxx --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=PrivateRouteTable}]'

# Add route to NAT Gateway
aws ec2 create-route --route-table-id rtb-xxxxxxxx --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-xxxxxxxx

# Associate with private subnet
aws ec2 associate-route-table --subnet-id subnet-xxxxxxxx --route-table-id rtb-xxxxxxxx
```

### Phase 4: Network Security Implementation

#### 4.1 Security Groups

**Web Server Security Group:**
```bash
# Create security group for web servers
aws ec2 create-security-group --group-name web-sg --description "Security group for web servers" --vpc-id vpc-xxxxxxxx

# Allow HTTP traffic
aws ec2 authorize-security-group-ingress --group-id sg-xxxxxxxx --protocol tcp --port 80 --cidr 0.0.0.0/0

# Allow HTTPS traffic
aws ec2 authorize-security-group-ingress --group-id sg-xxxxxxxx --protocol tcp --port 443 --cidr 0.0.0.0/0

# Allow SSH from your IP
aws ec2 authorize-security-group-ingress --group-id sg-xxxxxxxx --protocol tcp --port 22 --cidr YOUR_IP/32
```

**Database Security Group:**
```bash
# Create security group for database servers
aws ec2 create-security-group --group-name db-sg --description "Security group for database servers" --vpc-id vpc-xxxxxxxx

# Allow MySQL/Aurora access from web security group
aws ec2 authorize-security-group-ingress --group-id sg-xxxxxxxx --protocol tcp --port 3306 --source-group sg-web-xxxxxxxx

# Allow SSH from bastion/web servers
aws ec2 authorize-security-group-ingress --group-id sg-xxxxxxxx --protocol tcp --port 22 --source-group sg-web-xxxxxxxx
```

#### 4.2 Network Access Control Lists (NACLs)

**Public Subnet NACL:**
```bash
# Create NACL for public subnet
aws ec2 create-network-acl --vpc-id vpc-xxxxxxxx --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=PublicNACL}]'

# Allow inbound HTTP
aws ec2 create-network-acl-entry --network-acl-id acl-xxxxxxxx --rule-number 100 --protocol tcp --port-range From=80,To=80 --cidr-block 0.0.0.0/0

# Allow inbound HTTPS
aws ec2 create-network-acl-entry --network-acl-id acl-xxxxxxxx --rule-number 110 --protocol tcp --port-range From=443,To=443 --cidr-block 0.0.0.0/0

# Allow SSH
aws ec2 create-network-acl-entry --network-acl-id acl-xxxxxxxx --rule-number 120 --protocol tcp --port-range From=22,To=22 --cidr-block YOUR_IP/32
```

### Phase 5: IAM Roles and Policies

#### 5.1 EC2 Instance Role
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

#### 5.2 Instance Profile Creation
```bash
# Create IAM role
aws iam create-role --role-name EC2-VPC-Role --assume-role-policy-document file://trust-policy.json

# Attach policies (example: CloudWatch logs)
aws iam attach-role-policy --role-name EC2-VPC-Role --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess

# Create instance profile
aws iam create-instance-profile --instance-profile-name EC2-VPC-Profile

# Add role to profile
aws iam add-role-to-instance-profile --instance-profile-name EC2-VPC-Profile --role-name EC2-VPC-Role
```

### Phase 6: SSH Key Pair Management

#### 6.1 Generate SSH Key Pair
```bash
# Generate new key pair
aws ec2 create-key-pair --key-name MyVPCKeyPair --query 'KeyMaterial' --output text > MyVPCKeyPair.pem

# Set proper permissions
chmod 400 MyVPCKeyPair.pem
```

#### 6.2 Secure Key Storage
- Store private key in secure location
- Never commit keys to version control
- Consider using AWS Systems Manager Parameter Store for additional security

### Phase 7: EC2 Instance Provisioning

#### 7.1 Launch Public Instance (Web Server)
```bash
aws ec2 run-instances \
  --image-id ami-xxxxxxxx \
  --count 1 \
  --instance-type t3.micro \
  --key-name MyVPCKeyPair \
  --security-group-ids sg-web-xxxxxxxx \
  --subnet-id subnet-public-xxxxxxxx \
  --iam-instance-profile Name=EC2-VPC-Profile \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=WebServer}]'
```

#### 7.2 Launch Private Instance (Database Server)
```bash
aws ec2 run-instances \
  --image-id ami-xxxxxxxx \
  --count 1 \
  --instance-type t3.micro \
  --key-name MyVPCKeyPair \
  --security-group-ids sg-db-xxxxxxxx \
  --subnet-id subnet-private-xxxxxxxx \
  --iam-instance-profile Name=EC2-VPC-Profile \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=DatabaseServer}]'
```

## Testing and Validation

### Test 1: SSH Access to Public Instance
```bash
# Connect to public instance
ssh -i MyVPCKeyPair.pem ec2-user@PUBLIC_IP

# Verify internet connectivity
curl -I http://www.google.com
```

### Test 2: SSH Access to Private Instance
```bash
# From public instance, connect to private instance
ssh -i MyVPCKeyPair.pem ec2-user@PRIVATE_IP

# Test outbound internet access (should work via NAT Gateway)
curl -I http://www.google.com
```

### Test 3: Security Group Validation
```bash
# Test allowed ports
telnet PUBLIC_IP 80
telnet PUBLIC_IP 443

# Test blocked ports (should fail)
telnet PUBLIC_IP 3306
```

### Test 4: Network Connectivity Between Subnets
```bash
# From public instance, ping private instance
ping PRIVATE_IP

# Test application-specific connectivity
telnet PRIVATE_IP 3306  # MySQL connection test
```

## Security Best Practices

### Network Security
- Use principle of least privilege for security groups
- Implement defense in depth with both Security Groups and NACLs
- Regular audit of security group rules
- Use specific IP ranges instead of 0.0.0.0/0 when possible

### Access Control
- Rotate SSH keys regularly
- Use IAM roles instead of access keys for EC2 instances
- Enable CloudTrail for audit logging
- Implement multi-factor authentication

### Monitoring and Logging
- Enable VPC Flow Logs
- Set up CloudWatch alarms for unusual network activity
- Monitor failed SSH attempts
- Use AWS Config for compliance monitoring

## Cost Optimization

### Resource Management
- Use appropriate instance types (t3.micro for testing)
- Stop instances when not needed
- Release unused Elastic IPs
- Monitor NAT Gateway data processing charges

### Automation
- Use Infrastructure as Code (CloudFormation/Terraform)
- Implement auto-scaling for production workloads
- Schedule instance start/stop for development environments

## Troubleshooting Guide

### Common Issues

#### 1. Cannot SSH to Public Instance
- Check security group allows SSH (port 22) from your IP
- Verify instance has public IP assigned
- Check NACL rules
- Ensure key pair permissions are correct (chmod 400)

#### 2. Private Instance Cannot Access Internet
- Verify NAT Gateway is running and properly configured
- Check private route table has route to NAT Gateway
- Confirm security group allows outbound traffic

#### 3. Communication Between Subnets Fails
- Check security group rules allow traffic between instances
- Verify NACL rules for both subnets
- Confirm route tables are properly configured

## Commands Reference

### Useful AWS CLI Commands
```bash
# List VPCs
aws ec2 describe-vpcs

# List subnets
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-xxxxxxxx"

# Check security groups
aws ec2 describe-security-groups --group-ids sg-xxxxxxxx

# Monitor instances
aws ec2 describe-instances --filters "Name=vpc-id,Values=vpc-xxxxxxxx"

# Check route tables
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-xxxxxxxx"
```

## Advanced Extensions

### Optional Enhancements
1. **Multi-AZ Deployment**: Deploy across multiple availability zones
2. **Load Balancer**: Add Application Load Balancer for high availability
3. **Auto Scaling**: Implement auto scaling groups
4. **Database Setup**: Install and configure MySQL/PostgreSQL
5. **Monitoring**: Set up comprehensive CloudWatch monitoring
6. **Backup Strategy**: Implement automated backup solutions

## Resources and Documentation

- [AWS VPC User Guide](https://docs.aws.amazon.com/vpc/)
- [EC2 Security Groups](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security-groups.html)
- [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [VPC Networking Components](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Networking.html)

## Conclusion

This project provides hands-on experience with fundamental AWS networking and security concepts. The secure VPC setup with proper subnet segmentation, security controls, and access management forms the foundation for scalable and secure cloud architectures.

Remember to clean up resources after testing to avoid unnecessary charges:
```bash
# Terminate instances
aws ec2 terminate-instances --instance-ids i-xxxxxxxx

# Delete NAT Gateway
aws ec2 delete-nat-gateway --nat-gateway-id nat-xxxxxxxx

# Release Elastic IP
aws ec2 release-address --allocation-id eipalloc-xxxxxxxx

# Delete VPC (will cascade delete subnets, route tables, etc.)
aws ec2 delete-vpc --vpc-id vpc-xxxxxxxx
```