# AWS EC2, IAM & VPC Scenario-Based Interview Guide

## Table of Contents
1. [2-Tier Application VPC Architecture](#1-2-tier-application-vpc-architecture)
2. [Restricting Outbound Internet Access](#2-restricting-outbound-internet-access)
3. [Private Subnet Internet Access via NAT](#3-private-subnet-internet-access-via-nat)
4. [Inter-Instance Communication in VPC](#4-inter-instance-communication-in-vpc)
5. [Network Access Control with NACLs](#5-network-access-control-with-nacls)
6. [Isolated Environment Setup](#6-isolated-environment-setup)
7. [Secure AWS Services Access with VPC Endpoints](#7-secure-aws-services-access-with-vpc-endpoints)
8. [NACLs vs Security Groups](#8-nacls-vs-security-groups)
9. [IAM Components Explained](#9-iam-components-explained)
10. [Bastion Host Setup](#10-bastion-host-setup)

---

## 1. 2-Tier Application VPC Architecture

### Scenario
Design a VPC architecture for a 2-tier application that needs to be highly available and scalable.

### Step-by-Step Implementation

#### Step 1: Create VPC
```bash
# Create VPC with CIDR block
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=MyApp-VPC}]'
```

#### Step 2: Create Subnets Across Multiple AZs
```bash
# Public Subnet in AZ-1a
aws ec2 create-subnet --vpc-id vpc-12345678 --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Public-Subnet-1a}]'

# Public Subnet in AZ-1b
aws ec2 create-subnet --vpc-id vpc-12345678 --cidr-block 10.0.2.0/24 --availability-zone us-east-1b --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Public-Subnet-1b}]'

# Private Subnet in AZ-1a
aws ec2 create-subnet --vpc-id vpc-12345678 --cidr-block 10.0.3.0/24 --availability-zone us-east-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Private-Subnet-1a}]'

# Private Subnet in AZ-1b
aws ec2 create-subnet --vpc-id vpc-12345678 --cidr-block 10.0.4.0/24 --availability-zone us-east-1b --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Private-Subnet-1b}]'
```

#### Step 3: Create and Attach Internet Gateway
```bash
# Create Internet Gateway
aws ec2 create-internet-gateway --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=MyApp-IGW}]'

# Attach to VPC
aws ec2 attach-internet-gateway --internet-gateway-id igw-12345678 --vpc-id vpc-12345678
```

#### Step 4: Create Route Tables
```bash
# Public Route Table
aws ec2 create-route-table --vpc-id vpc-12345678 --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Public-RT}]'

# Add route to Internet Gateway
aws ec2 create-route --route-table-id rtb-12345678 --destination-cidr-block 0.0.0.0/0 --gateway-id igw-12345678

# Associate public subnets with public route table
aws ec2 associate-route-table --subnet-id subnet-12345678 --route-table-id rtb-12345678
aws ec2 associate-route-table --subnet-id subnet-87654321 --route-table-id rtb-12345678
```

#### Step 5: Create Security Groups
```bash
# Load Balancer Security Group (Public Tier)
aws ec2 create-security-group --group-name ALB-SG --description "Load Balancer Security Group" --vpc-id vpc-12345678

# Allow HTTP/HTTPS from anywhere
aws ec2 authorize-security-group-ingress --group-id sg-12345678 --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id sg-12345678 --protocol tcp --port 443 --cidr 0.0.0.0/0

# Application Server Security Group (Private Tier)
aws ec2 create-security-group --group-name App-SG --description "Application Server Security Group" --vpc-id vpc-12345678

# Allow traffic from Load Balancer only
aws ec2 authorize-security-group-ingress --group-id sg-87654321 --protocol tcp --port 8080 --source-group sg-12345678
```

#### Step 6: Deploy Application Load Balancer
```bash
# Create Application Load Balancer in public subnets
aws elbv2 create-load-balancer --name MyApp-ALB --subnets subnet-12345678 subnet-87654321 --security-groups sg-12345678
```

#### Step 7: Create Auto Scaling Group
```bash
# Create Launch Template
aws ec2 create-launch-template --launch-template-name MyApp-Template --launch-template-data '{
  "ImageId": "ami-12345678",
  "InstanceType": "t3.medium",
  "SecurityGroupIds": ["sg-87654321"],
  "UserData": "base64-encoded-script"
}'

# Create Auto Scaling Group
aws autoscaling create-auto-scaling-group --auto-scaling-group-name MyApp-ASG --launch-template LaunchTemplateName=MyApp-Template,Version=1 --min-size 2 --max-size 6 --desired-capacity 2 --vpc-zone-identifier "subnet-13579246,subnet-97531864"
```

---

## 2. Restricting Outbound Internet Access

### Scenario
Restrict outbound internet access for resources in one subnet while allowing it for another subnet.

### Step-by-Step Implementation

#### Step 1: Identify Current Route Tables
```bash
# List all route tables in VPC
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-12345678"
```

#### Step 2: Create Restricted Route Table
```bash
# Create new route table for restricted subnet
aws ec2 create-route-table --vpc-id vpc-12345678 --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Restricted-RT}]'
```

#### Step 3: Remove Internet Gateway Route
```bash
# Do NOT add default route (0.0.0.0/0) to Internet Gateway
# Only keep local VPC routes (automatically added)
```

#### Step 4: Associate Restricted Subnet
```bash
# Associate restricted subnet with the new route table
aws ec2 associate-route-table --subnet-id subnet-restricted --route-table-id rtb-restricted
```

#### Step 5: Verify Configuration
```bash
# Check route table associations
aws ec2 describe-route-tables --route-table-ids rtb-restricted
```

**Result**: Instances in the restricted subnet can only communicate within the VPC, while instances in other subnets maintain internet access.

---

## 3. Private Subnet Internet Access via NAT

### Scenario
Allow instances in private subnet to access internet for software updates while maintaining security.

### Step-by-Step Implementation

#### Step 1: Create NAT Gateway
```bash
# Allocate Elastic IP for NAT Gateway
aws ec2 allocate-address --domain vpc

# Create NAT Gateway in public subnet
aws ec2 create-nat-gateway --subnet-id subnet-public --allocation-id eipalloc-12345678 --tag-specifications 'ResourceType=nat-gateway,Tags=[{Key=Name,Value=MyApp-NAT}]'
```

#### Step 2: Create Private Route Table
```bash
# Create route table for private subnet
aws ec2 create-route-table --vpc-id vpc-12345678 --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Private-RT}]'
```

#### Step 3: Add NAT Gateway Route
```bash
# Add route to NAT Gateway for internet access
aws ec2 create-route --route-table-id rtb-private --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-12345678
```

#### Step 4: Associate Private Subnet
```bash
# Associate private subnet with private route table
aws ec2 associate-route-table --subnet-id subnet-private --route-table-id rtb-private
```

#### Step 5: Test Connectivity
```bash
# From private instance, test internet connectivity
curl -I http://google.com
```

**Benefits**:
- Outbound internet access for updates
- No inbound internet access (security maintained)
- Centralized internet access control

---

## 4. Inter-Instance Communication in VPC

### Scenario
Enable EC2 instances to communicate with each other using private IP addresses within VPC.

### Step-by-Step Implementation

#### Step 1: Verify VPC Configuration
```bash
# Check if instances are in same VPC
aws ec2 describe-instances --instance-ids i-12345678 i-87654321 --query 'Reservations[].Instances[].[InstanceId,VpcId,SubnetId,PrivateIpAddress]'
```

#### Step 2: Configure Security Groups
```bash
# Create security group allowing internal communication
aws ec2 create-security-group --group-name Internal-Communication-SG --description "Allow internal VPC communication" --vpc-id vpc-12345678

# Allow all traffic from same security group
aws ec2 authorize-security-group-ingress --group-id sg-internal --protocol -1 --source-group sg-internal

# Or allow specific ports (e.g., SSH, HTTP)
aws ec2 authorize-security-group-ingress --group-id sg-internal --protocol tcp --port 22 --source-group sg-internal
aws ec2 authorize-security-group-ingress --group-id sg-internal --protocol tcp --port 80 --source-group sg-internal
```

#### Step 3: Apply Security Group to Instances
```bash
# Modify instance security groups
aws ec2 modify-instance-attribute --instance-id i-12345678 --groups sg-internal
aws ec2 modify-instance-attribute --instance-id i-87654321 --groups sg-internal
```

#### Step 4: Test Communication
```bash
# From instance 1, ping instance 2 using private IP
ping 10.0.1.100

# SSH to another instance
ssh -i keypair.pem ec2-user@10.0.1.100
```

#### Step 5: Verify Network ACLs
```bash
# Check NACL rules (should allow internal traffic by default)
aws ec2 describe-network-acls --filters "Name=vpc-id,Values=vpc-12345678"
```

---

## 5. Network Access Control with NACLs

### Scenario
Implement strict network access control for VPC resources using Network Access Control Lists.

### Step-by-Step Implementation

#### Step 1: Create Custom NACL
```bash
# Create new Network ACL
aws ec2 create-network-acl --vpc-id vpc-12345678 --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=Strict-NACL}]'
```

#### Step 2: Configure Inbound Rules
```bash
# Allow HTTP traffic (port 80)
aws ec2 create-network-acl-entry --network-acl-id acl-12345678 --rule-number 100 --protocol tcp --rule-action allow --port-range From=80,To=80 --cidr-block 0.0.0.0/0

# Allow HTTPS traffic (port 443)
aws ec2 create-network-acl-entry --network-acl-id acl-12345678 --rule-number 110 --protocol tcp --rule-action allow --port-range From=443,To=443 --cidr-block 0.0.0.0/0

# Allow SSH from specific IP range
aws ec2 create-network-acl-entry --network-acl-id acl-12345678 --rule-number 120 --protocol tcp --rule-action allow --port-range From=22,To=22 --cidr-block 203.0.113.0/24

# Deny all other inbound traffic (implicit, but can be explicit)
aws ec2 create-network-acl-entry --network-acl-id acl-12345678 --rule-number 32766 --protocol -1 --rule-action deny --cidr-block 0.0.0.0/0
```

#### Step 3: Configure Outbound Rules
```bash
# Allow HTTP responses (ephemeral ports)
aws ec2 create-network-acl-entry --network-acl-id acl-12345678 --rule-number 100 --protocol tcp --rule-action allow --port-range From=1024,To=65535 --cidr-block 0.0.0.0/0 --egress

# Allow HTTPS outbound
aws ec2 create-network-acl-entry --network-acl-id acl-12345678 --rule-number 110 --protocol tcp --rule-action allow --port-range From=443,To=443 --cidr-block 0.0.0.0/0 --egress

# Allow DNS queries
aws ec2 create-network-acl-entry --network-acl-id acl-12345678 --rule-number 120 --protocol udp --rule-action allow --port-range From=53,To=53 --cidr-block 0.0.0.0/0 --egress
```

#### Step 4: Associate NACL with Subnet
```bash
# Associate NACL with target subnet
aws ec2 associate-network-acl --network-acl-id acl-12345678 --subnet-id subnet-12345678
```

#### Step 5: Monitor and Test
```bash
# Test connectivity from allowed and denied sources
# Monitor VPC Flow Logs for blocked traffic
aws logs filter-log-events --log-group-name VPCFlowLogs --filter-pattern "REJECT"
```

---

## 6. Isolated Environment Setup

### Scenario
Create an isolated environment within VPC for sensitive workloads with no direct internet connectivity.

### Step-by-Step Implementation

#### Step 1: Create Isolated Subnet
```bash
# Create isolated subnet
aws ec2 create-subnet --vpc-id vpc-12345678 --cidr-block 10.0.5.0/24 --availability-zone us-east-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Isolated-Subnet}]'
```

#### Step 2: Create Isolated Route Table
```bash
# Create route table with NO internet gateway route
aws ec2 create-route-table --vpc-id vpc-12345678 --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Isolated-RT}]'

# Only local VPC route exists (automatically added)
# DO NOT add 0.0.0.0/0 route to IGW
```

#### Step 3: Associate Isolated Subnet
```bash
# Associate isolated subnet with isolated route table
aws ec2 associate-route-table --subnet-id subnet-isolated --route-table-id rtb-isolated
```

#### Step 4: Create Restrictive Security Group
```bash
# Create security group for isolated workloads
aws ec2 create-security-group --group-name Isolated-SG --description "Isolated workload security group" --vpc-id vpc-12345678

# Allow only necessary internal communication
aws ec2 authorize-security-group-ingress --group-id sg-isolated --protocol tcp --port 443 --cidr 10.0.0.0/16
aws ec2 authorize-security-group-ingress --group-id sg-isolated --protocol tcp --port 22 --cidr 10.0.0.0/16
```

#### Step 5: Optional - Add NAT Gateway for Updates
```bash
# If outbound internet access needed for updates
aws ec2 create-route --route-table-id rtb-isolated --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-12345678
```

#### Step 6: Deploy Isolated Workloads
```bash
# Launch instances in isolated subnet
aws ec2 run-instances --image-id ami-12345678 --instance-type t3.medium --subnet-id subnet-isolated --security-group-ids sg-isolated --key-name my-keypair
```

---

## 7. Secure AWS Services Access with VPC Endpoints

### Scenario
Enable secure access to AWS services (like S3) from within VPC without internet gateway dependency.

### Step-by-Step Implementation

#### Step 1: Create S3 VPC Endpoint (Gateway)
```bash
# Create VPC endpoint for S3
aws ec2 create-vpc-endpoint --vpc-id vpc-12345678 --service-name com.amazonaws.us-east-1.s3 --vpc-endpoint-type Gateway --route-table-ids rtb-12345678 rtb-87654321 --tag-specifications 'ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=S3-Endpoint}]'
```

#### Step 2: Create DynamoDB VPC Endpoint (Gateway)
```bash
# Create VPC endpoint for DynamoDB
aws ec2 create-vpc-endpoint --vpc-id vpc-12345678 --service-name com.amazonaws.us-east-1.dynamodb --vpc-endpoint-type Gateway --route-table-ids rtb-12345678
```

#### Step 3: Create Interface Endpoints (for other services)
```bash
# Create interface endpoint for EC2 service
aws ec2 create-vpc-endpoint --vpc-id vpc-12345678 --service-name com.amazonaws.us-east-1.ec2 --vpc-endpoint-type Interface --subnet-ids subnet-12345678 --security-group-ids sg-endpoint --private-dns-enabled

# Security group for interface endpoints
aws ec2 create-security-group --group-name VPC-Endpoint-SG --description "VPC Interface Endpoint Security Group" --vpc-id vpc-12345678
aws ec2 authorize-security-group-ingress --group-id sg-endpoint --protocol tcp --port 443 --cidr 10.0.0.0/16
```

#### Step 4: Configure Endpoint Policies
```bash
# Create endpoint policy (JSON file)
cat > s3-endpoint-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::my-secure-bucket/*"
      ]
    }
  ]
}
EOF

# Apply policy to endpoint
aws ec2 modify-vpc-endpoint --vpc-endpoint-id vpce-12345678 --policy-document file://s3-endpoint-policy.json
```

#### Step 5: Test VPC Endpoint Access
```bash
# From EC2 instance, test S3 access via VPC endpoint
aws s3 ls s3://my-secure-bucket --region us-east-1

# Verify traffic goes through VPC endpoint (check VPC Flow Logs)
```

**Benefits**:
- Private connectivity to AWS services
- No internet gateway required
- Traffic stays within AWS network
- Fine-grained access control via policies

---

## 8. NACLs vs Security Groups

### Comparison Table

| Feature | Network ACLs (NACLs) | Security Groups |
|---------|---------------------|-----------------|
| **Level** | Subnet level | Instance level |
| **State** | Stateless | Stateful |
| **Rules** | Allow and Deny | Allow only |
| **Processing** | Process rules in order | Process all rules |
| **Default** | Allow all traffic | Deny all traffic |

### Use Case Example: Multi-Layer Security

#### Step 1: Configure NACL (Subnet Level)
```bash
# Create NACL for web tier
aws ec2 create-network-acl --vpc-id vpc-12345678 --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=Web-Tier-NACL}]'

# Allow HTTP/HTTPS inbound
aws ec2 create-network-acl-entry --network-acl-id acl-web --rule-number 100 --protocol tcp --rule-action allow --port-range From=80,To=80 --cidr-block 0.0.0.0/0
aws ec2 create-network-acl-entry --network-acl-id acl-web --rule-number 110 --protocol tcp --rule-action allow --port-range From=443,To=443 --cidr-block 0.0.0.0/0

# Deny direct database access from internet
aws ec2 create-network-acl-entry --network-acl-id acl-web --rule-number 200 --protocol tcp --rule-action deny --port-range From=3306,To=3306 --cidr-block 0.0.0.0/0
```

#### Step 2: Configure Security Groups (Instance Level)
```bash
# Web server security group
aws ec2 create-security-group --group-name Web-SG --description "Web server security group" --vpc-id vpc-12345678
aws ec2 authorize-security-group-ingress --group-id sg-web --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id sg-web --protocol tcp --port 443 --cidr 0.0.0.0/0

# Database security group (only from web servers)
aws ec2 create-security-group --group-name DB-SG --description "Database security group" --vpc-id vpc-12345678
aws ec2 authorize-security-group-ingress --group-id sg-db --protocol tcp --port 3306 --source-group sg-web
```

#### Step 3: Apply Both Layers
```bash
# Associate NACL with web subnet
aws ec2 associate-network-acl --network-acl-id acl-web --subnet-id subnet-web

# Launch instances with appropriate security groups
aws ec2 run-instances --image-id ami-12345678 --security-group-ids sg-web --subnet-id subnet-web
aws ec2 run-instances --image-id ami-database --security-group-ids sg-db --subnet-id subnet-db
```

---

## 9. IAM Components Explained

### IAM Users

#### Step 1: Create IAM User
```bash
# Create user
aws iam create-user --user-name john-developer

# Create access keys
aws iam create-access-key --user-name john-developer
```

#### Step 2: Attach Policy to User
```bash
# Attach managed policy
aws iam attach-user-policy --user-name john-developer --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Create and attach custom policy
aws iam create-policy --policy-name CustomS3Policy --policy-document file://custom-policy.json
aws iam attach-user-policy --user-name john-developer --policy-arn arn:aws:iam::123456789012:policy/CustomS3Policy
```

### IAM Groups

#### Step 1: Create IAM Group
```bash
# Create group
aws iam create-group --group-name Developers

# Attach policy to group
aws iam attach-group-policy --group-name Developers --policy-arn arn:aws:iam::aws:policy/PowerUserAccess
```

#### Step 2: Add Users to Group
```bash
# Add user to group
aws iam add-user-to-group --user-name john-developer --group-name Developers
aws iam add-user-to-group --user-name jane-developer --group-name Developers
```

### IAM Roles

#### Step 1: Create IAM Role
```bash
# Create trust policy for EC2
cat > trust-policy.json << EOF
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
EOF

# Create role
aws iam create-role --role-name EC2-S3-Access-Role --assume-role-policy-document file://trust-policy.json
```

#### Step 2: Attach Policies to Role
```bash
# Attach managed policy
aws iam attach-role-policy --role-name EC2-S3-Access-Role --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

# Create instance profile
aws iam create-instance-profile --instance-profile-name EC2-Profile
aws iam add-role-to-instance-profile --instance-profile-name EC2-Profile --role-name EC2-S3-Access-Role
```

#### Step 3: Use Role with EC2
```bash
# Launch EC2 with IAM role
aws ec2 run-instances --image-id ami-12345678 --instance-type t3.micro --iam-instance-profile Name=EC2-Profile
```

### IAM Policies

#### Example Custom Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*"
    },
    {
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "*"
    }
  ]
}
```

---

## 10. Bastion Host Setup

### Scenario
Set up secure access to private subnet instances through a bastion host.

### Step-by-Step Implementation

#### Step 1: Launch Bastion Host in Public Subnet
```bash
# Create bastion host security group
aws ec2 create-security-group --group-name Bastion-SG --description "Bastion Host Security Group" --vpc-id vpc-12345678

# Allow SSH from your IP only
aws ec2 authorize-security-group-ingress --group-id sg-bastion --protocol tcp --port 22 --cidr YOUR_IP/32

# Launch bastion host
aws ec2 run-instances \
  --image-id ami-12345678 \
  --instance-type t3.micro \
  --key-name my-keypair \
  --subnet-id subnet-public \
  --security-group-ids sg-bastion \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Bastion-Host}]'
```

#### Step 2: Allocate and Associate Elastic IP
```bash
# Allocate Elastic IP
aws ec2 allocate-address --domain vpc

# Associate with bastion host
aws ec2 associate-address --instance-id i-bastion --allocation-id eipalloc-12345678
```

#### Step 3: Configure Private Instance Security Group
```bash
# Create private instance security group
aws ec2 create-security-group --group-name Private-SG --description "Private Instance Security Group" --vpc-id vpc-12345678

# Allow SSH from bastion host only
aws ec2 authorize-security-group-ingress --group-id sg-private --protocol tcp --port 22 --source-group sg-bastion

# Allow application traffic from ALB
aws ec2 authorize-security-group-ingress --group-id sg-private --protocol tcp --port 8080 --source-group sg-alb
```

#### Step 4: Launch Private Instances
```bash
# Launch instances in private subnet
aws ec2 run-instances \
  --image-id ami-12345678 \
  --instance-type t3.medium \
  --key-name my-keypair \
  --subnet-id subnet-private \
  --security-group-ids sg-private \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Private-App-Server}]'
```

#### Step 5: Configure SSH Agent Forwarding
```bash
# On your local machine, add SSH key to agent
ssh-add ~/.ssh/my-keypair.pem

# Connect to bastion with agent forwarding
ssh -A ec2-user@BASTION_PUBLIC_IP

# From bastion, connect to private instance
ssh ec2-user@PRIVATE_INSTANCE_IP
```

#### Step 6: Alternative - SSH Tunneling
```bash
# Direct tunnel to private instance through bastion
ssh -i ~/.ssh/my-keypair.pem -J ec2-user@BASTION_PUBLIC_IP ec2-user@PRIVATE_INSTANCE_IP

# Or create tunnel for specific service
ssh -i ~/.ssh/my-keypair.pem -L 8080:PRIVATE_INSTANCE_IP:8080 ec2-user@BASTION_PUBLIC_IP
```

#### Step 7: Harden Bastion Host
```bash
# Connect to bastion and configure
sudo yum update -y

# Configure SSH (edit /etc/ssh/sshd_config)
sudo nano /etc/ssh/sshd_config

# Add these settings:
# PermitRootLogin no
# PasswordAuthentication no
# X11Forwarding no
# MaxAuthTries 3
# ClientAliveInterval 300
# ClientAliveCountMax 2

# Restart SSH service
sudo systemctl restart sshd

# Configure fail2ban for additional security
sudo yum install epel-release -y
sudo yum install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

#### Step 8: Monitor and Log Access
```bash
# Enable VPC Flow Logs
aws ec2 create-flow-logs \
  --resource-type Instance \
  --resource-ids i-bastion \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name BastionFlowLogs

# Monitor SSH access logs
sudo tail -f /var/log/secure
```

---

## Best Practices Summary

### VPC Design
- Use multiple AZs for high availability
- Implement layered security (NACLs + Security Groups)
- Use private subnets for application tiers
- Implement least privilege access

### Security
- Regular security group audits
- Enable VPC Flow Logs
- Use IAM roles instead of access keys where possible
- Implement MFA for critical access

### Monitoring
- Set up CloudWatch alarms
- Monitor VPC Flow Logs
- Use AWS Config for compliance
- Implement centralized logging

### Cost Optimization
- Use NAT instances instead of NAT Gateways for low traffic
- Right-size your instances
- Use Spot instances where appropriate
- Monitor data transfer costs

---

## 11. Advanced Scenarios and Troubleshooting

### Multi-Account VPC Peering

#### Scenario
Connect VPCs across different AWS accounts for resource sharing.

#### Step-by-Step Implementation

##### Step 1: Create VPC Peering Connection (Account A)
```bash
# From Account A, create peering connection to Account B
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-12345678 \
  --peer-vpc-id vpc-87654321 \
  --peer-owner-id 123456789012 \
  --peer-region us-east-1
```

##### Step 2: Accept Peering Connection (Account B)
```bash
# From Account B, accept the peering connection
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id pcx-12345678
```

##### Step 3: Update Route Tables (Both Accounts)
```bash
# Account A - Add route to Account B VPC
aws ec2 create-route \
  --route-table-id rtb-account-a \
  --destination-cidr-block 172.16.0.0/16 \
  --vpc-peering-connection-id pcx-12345678

# Account B - Add route to Account A VPC
aws ec2 create-route \
  --route-table-id rtb-account-b \
  --destination-cidr-block 10.0.0.0/16 \
  --vpc-peering-connection-id pcx-12345678
```

##### Step 4: Update Security Groups
```bash
# Allow traffic from peer VPC CIDR
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 443 \
  --cidr 172.16.0.0/16
```

### Transit Gateway Implementation

#### Scenario
Create hub-and-spoke network architecture for multiple VPCs.

#### Step-by-Step Implementation

##### Step 1: Create Transit Gateway
```bash
# Create Transit Gateway
aws ec2 create-transit-gateway \
  --description "Central Hub for VPC Connectivity" \
  --options DefaultRouteTableAssociation=enable,DefaultRouteTablePropagation=enable \
  --tag-specifications 'ResourceType=transit-gateway,Tags=[{Key=Name,Value=Central-TGW}]'
```

##### Step 2: Attach VPCs to Transit Gateway
```bash
# Attach VPC 1
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id tgw-12345678 \
  --vpc-id vpc-11111111 \
  --subnet-ids subnet-11111111

# Attach VPC 2
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id tgw-12345678 \
  --vpc-id vpc-22222222 \
  --subnet-ids subnet-22222222
```

##### Step 3: Configure Route Tables
```bash
# Create custom route table for specific routing
aws ec2 create-transit-gateway-route-table --transit-gateway-id tgw-12345678

# Associate attachment with custom route table
aws ec2 associate-transit-gateway-route-table \
  --transit-gateway-attachment-id tgw-attach-12345678 \
  --transit-gateway-route-table-id tgw-rtb-12345678
```

### Site-to-Site VPN Configuration

#### Scenario
Connect on-premises network to AWS VPC securely.

#### Step-by-Step Implementation

##### Step 1: Create Customer Gateway
```bash
# Create customer gateway (represents your on-premises router)
aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --public-ip YOUR_PUBLIC_IP \
  --bgp-asn 65000 \
  --tag-specifications 'ResourceType=customer-gateway,Tags=[{Key=Name,Value=OnPrem-CGW}]'
```

##### Step 2: Create Virtual Private Gateway
```bash
# Create VPN gateway
aws ec2 create-vpn-gateway \
  --type ipsec.1 \
  --tag-specifications 'ResourceType=vpn-gateway,Tags=[{Key=Name,Value=AWS-VGW}]'

# Attach to VPC
aws ec2 attach-vpn-gateway --vpn-gateway-id vgw-12345678 --vpc-id vpc-12345678
```

##### Step 3: Create VPN Connection
```bash
# Create Site-to-Site VPN connection
aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id cgw-12345678 \
  --vpn-gateway-id vgw-12345678 \
  --options StaticRoutesOnly=true
```

##### Step 4: Configure Static Routes
```bash
# Add static route for on-premises network
aws ec2 create-vpn-connection-route \
  --vpn-connection-id vpn-12345678 \
  --destination-cidr-block 192.168.0.0/16
```

##### Step 5: Update VPC Route Tables
```bash
# Enable route propagation
aws ec2 enable-vgw-route-propagation \
  --route-table-id rtb-12345678 \
  --gateway-id vgw-12345678
```

## 12. Security Deep Dive

### AWS Systems Manager Session Manager Setup

#### Scenario
Enable secure shell access without SSH keys or bastion hosts.

#### Step-by-Step Implementation

##### Step 1: Create IAM Role for EC2
```bash
# Create trust policy
cat > ssm-trust-policy.json << EOF
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
EOF

# Create role
aws iam create-role \
  --role-name EC2-SSM-Role \
  --assume-role-policy-document file://ssm-trust-policy.json
```

##### Step 2: Attach Required Policies
```bash
# Attach managed policies
aws iam attach-role-policy \
  --role-name EC2-SSM-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam attach-role-policy \
  --role-name EC2-SSM-Role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
```

##### Step 3: Create Instance Profile
```bash
# Create instance profile
aws iam create-instance-profile --instance-profile-name EC2-SSM-Profile

# Add role to instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-SSM-Profile \
  --role-name EC2-SSM-Role
```

##### Step 4: Create VPC Endpoint for SSM
```bash
# Create VPC endpoints for Session Manager
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.us-east-1.ssm \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-12345678 \
  --security-group-ids sg-ssm-endpoint

aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.us-east-1.ssmmessages \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-12345678 \
  --security-group-ids sg-ssm-endpoint

aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.us-east-1.ec2messages \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-12345678 \
  --security-group-ids sg-ssm-endpoint
```

##### Step 5: Launch Instance with SSM Role
```bash
# Launch instance with SSM role
aws ec2 run-instances \
  --image-id ami-12345678 \
  --instance-type t3.micro \
  --iam-instance-profile Name=EC2-SSM-Profile \
  --subnet-id subnet-private \
  --security-group-ids sg-private \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=SSM-Managed-Instance}]'
```

##### Step 6: Connect via Session Manager
```bash
# Start session (no SSH key required)
aws ssm start-session --target i-1234567890abcdef0

# Or use AWS CLI v2 with session manager plugin
aws ssm start-session --target i-1234567890abcdef0 --region us-east-1
```

### AWS GuardDuty Integration

#### Step 1: Enable GuardDuty
```bash
# Enable GuardDuty
aws guardduty create-detector --enable --finding-publishing-frequency FIFTEEN_MINUTES
```

#### Step 2: Create Custom Threat Intelligence
```bash
# Upload threat intelligence list
aws guardduty create-threat-intel-set \
  --detector-id 12abc34d567e8f4912ab34c56de78f90 \
  --name "CustomThreatIntel" \
  --format TXT \
  --location s3://my-threat-intel-bucket/threat-list.txt \
  --activate
```

#### Step 3: Configure Event Notifications
```bash
# Create SNS topic for GuardDuty findings
aws sns create-topic --name guardduty-findings

# Create EventBridge rule
aws events put-rule \
  --name GuardDutyFindingRule \
  --event-pattern '{
    "source": ["aws.guardduty"],
    "detail-type": ["GuardDuty Finding"],
    "detail": {
      "severity": [7.0, 7.1, 7.2, 7.3, 7.4, 7.5, 7.6, 7.7, 7.8, 7.9, 8.0, 8.1, 8.2, 8.3, 8.4, 8.5, 8.6, 8.7, 8.8, 8.9, 9.0, 9.1, 9.2, 9.3, 9.4, 9.5, 9.6, 9.7, 9.8, 9.9, 10.0]
    }
  }'
```

## 13. Performance and Monitoring

### VPC Flow Logs Analysis

#### Step 1: Enable VPC Flow Logs
```bash
# Enable VPC Flow Logs to CloudWatch
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-12345678 \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name VPCFlowLogs \
  --deliver-logs-permission-arn arn:aws:iam::123456789012:role/flowlogsRole
```

#### Step 2: Create Custom Metrics
```bash
# Create metric filter for rejected connections
aws logs put-metric-filter \
  --log-group-name VPCFlowLogs \
  --filter-name RejectedConnections \
  --filter-pattern '[timestamp, account, eni, source, destination, srcport, destport="22", protocol="6", packets, bytes, windowstart, windowend, action="REJECT", flowlogstatus]' \
  --metric-transformations \
    metricName=SSHRejectedConnections,metricNamespace=VPC/Security,metricValue=1
```

#### Step 3: Create CloudWatch Alarms
```bash
# Create alarm for high rejected connections
aws cloudwatch put-metric-alarm \
  --alarm-name "High-SSH-Rejections" \
  --alarm-description "Alert when SSH connections are being rejected" \
  --metric-name SSHRejectedConnections \
  --namespace VPC/Security \
  --statistic Sum \
  --period 300 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2
```

### Enhanced Monitoring with CloudWatch Insights

#### Query Examples for VPC Flow Logs
```sql
-- Top 10 source IPs with most rejected connections
fields @timestamp, srcaddr, dstaddr, action
| filter action = "REJECT"
| stats count() as rejected_count by srcaddr
| sort rejected_count desc
| limit 10

-- Traffic analysis by protocol
fields @timestamp, protocol, srcport, dstport
| filter action = "ACCEPT"
| stats count() as connection_count by protocol
| sort connection_count desc

-- Identify unusual port access patterns
fields @timestamp, srcaddr, dstaddr, dstport
| filter action = "ACCEPT" and dstport not in [80, 443, 22]
| stats count() as unusual_connections by dstport, srcaddr
| sort unusual_connections desc
```

## 14. Disaster Recovery and Backup

### Cross-Region VPC Replication

#### Step 1: Create VPC in Secondary Region
```bash
# Switch to DR region
export AWS_DEFAULT_REGION=us-west-2

# Create identical VPC structure
aws ec2 create-vpc --cidr-block 10.1.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=DR-VPC}]'

# Create subnets in different AZs
aws ec2 create-subnet --vpc-id vpc-dr-12345678 --cidr-block 10.1.1.0/24 --availability-zone us-west-2a
aws ec2 create-subnet --vpc-id vpc-dr-12345678 --cidr-block 10.1.2.0/24 --availability-zone us-west-2b
```

#### Step 2: Set Up Inter-Region VPC Peering
```bash
# Create inter-region peering connection
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-12345678 \
  --peer-vpc-id vpc-dr-12345678 \
  --peer-region us-west-2
```

#### Step 3: Configure Route 53 Health Checks
```bash
# Create health check for primary region
aws route53 create-health-check \
  --caller-reference $(date +%s) \
  --health-check-config Type=HTTP,ResourcePath=/health,FullyQualifiedDomainName=primary.example.com,Port=80
```

### Automated AMI Backup Strategy

#### Step 1: Create IAM Role for Lambda
```bash
# Create Lambda execution role for AMI backups
cat > lambda-ami-backup-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateImage",
        "ec2:CreateSnapshot",
        "ec2:DescribeImages",
        "ec2:DescribeInstances",
        "ec2:DeregisterImage",
        "ec2:DeleteSnapshot",
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy --policy-name AMIBackupPolicy --policy-document file://lambda-ami-backup-policy.json
```

#### Step 2: Create Lambda Function for Automated Backups
```python
# ami-backup-lambda.py
import boto3
import json
from datetime import datetime, timedelta

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')
    
    # Get all running instances with backup tag
    instances = ec2.describe_instances(
        Filters=[
            {'Name': 'instance-state-name', 'Values': ['running']},
            {'Name': 'tag:AutoBackup', 'Values': ['true']}
        ]
    )
    
    for reservation in instances['Reservations']:
        for instance in reservation['Instances']:
            instance_id = instance['InstanceId']
            instance_name = next((tag['Value'] for tag in instance.get('Tags', []) if tag['Key'] == 'Name'), instance_id)
            
            # Create AMI
            ami_name = f"{instance_name}-backup-{datetime.now().strftime('%Y-%m-%d-%H-%M-%S')}"
            
            response = ec2.create_image(
                InstanceId=instance_id,
                Name=ami_name,
                Description=f"Automated backup of {instance_name}",
                NoReboot=True
            )
            
            # Tag the AMI
            ec2.create_tags(
                Resources=[response['ImageId']],
                Tags=[
                    {'Key': 'Name', 'Value': ami_name},
                    {'Key': 'AutoBackup', 'Value': 'true'},
                    {'Key': 'InstanceId', 'Value': instance_id},
                    {'Key': 'CreatedDate', 'Value': datetime.now().isoformat()}
                ]
            )
    
    # Cleanup old AMIs (older than 7 days)
    old_amis = ec2.describe_images(
        Owners=['self'],
        Filters=[
            {'Name': 'tag:AutoBackup', 'Values': ['true']},
            {'Name': 'creation-date', 'Values': [f"*{(datetime.now() - timedelta(days=7)).strftime('%Y-%m-%d')}*"]}
        ]
    )
    
    for ami in old_amis['Images']:
        # Deregister AMI and delete associated snapshots
        ec2.deregister_image(ImageId=ami['ImageId'])
        for block_device in ami['BlockDeviceMappings']:
            if 'Ebs' in block_device:
                ec2.delete_snapshot(SnapshotId=block_device['Ebs']['SnapshotId'])
    
    return {
        'statusCode': 200,
        'body': json.dumps('Backup process completed successfully')
    }
```

#### Step 3: Schedule with EventBridge
```bash
# Create EventBridge rule for daily backups
aws events put-rule \
  --name DailyAMIBackup \
  --schedule-expression "cron(0 2 * * ? *)" \
  --description "Daily AMI backup at 2 AM UTC"

# Add Lambda target to the rule
aws events put-targets \
  --rule DailyAMIBackup \
  --targets "Id"="1","Arn"="arn:aws:lambda:us-east-1:123456789012:function:AMIBackupFunction"
```

## 15. Troubleshooting Common Issues

### Network Connectivity Troubleshooting

#### Issue 1: Cannot Connect to Instance in Private Subnet

##### Diagnostic Steps:
```bash
# Check security group rules
aws ec2 describe-security-groups --group-ids sg-12345678

# Check NACL rules
aws ec2 describe-network-acls --filters "Name=association.subnet-id,Values=subnet-12345678"

# Check route table
aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=subnet-12345678"

# Check VPC endpoints
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=vpc-12345678"
```

##### Common Solutions:
```bash
# Add security group rule for bastion access
aws ec2 authorize-security-group-ingress --group-id sg-private --protocol tcp --port 22 --source-group sg-bastion

# Check if instance has SSM agent and proper IAM role
aws ssm describe-instance-information --filters "Key=InstanceIds,Values=i-12345678"
```

#### Issue 2: Internet Access Not Working Through NAT Gateway

##### Diagnostic Steps:
```bash
# Check NAT Gateway status
aws ec2 describe-nat-gateways --nat-gateway-ids nat-12345678

# Verify route table has correct route
aws ec2 describe-route-tables --route-table-ids rtb-private

# Check if NAT Gateway has Elastic IP
aws ec2 describe-addresses --filters "Name=association-id,Values=eipassoc-12345678"
```

##### Fix Route Table:
```bash
# Add missing route to NAT Gateway
aws ec2 create-route --route-table-id rtb-private --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-12345678
```

#### Issue 3: VPC Peering Connection Not Working

##### Diagnostic Steps:
```bash
# Check peering connection status
aws ec2 describe-vpc-peering-connections --vpc-peering-connection-ids pcx-12345678

# Verify route tables in both VPCs
aws ec2 describe-route-tables --filters "Name=route.destination-cidr-block,Values=PEER_VPC_CIDR"

# Check security groups allow cross-VPC traffic
aws ec2 describe-security-groups --group-ids sg-12345678
```

### Performance Optimization

#### Issue: High Data Transfer Costs

##### Analysis and Solutions:
```bash
# Enable VPC Flow Logs for cost analysis
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-12345678 \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination s3://vpc-flow-logs-bucket/

# Use CloudWatch Insights to analyze traffic patterns
# Query for inter-AZ traffic:
fields @timestamp, srcaddr, dstaddr, bytes
| filter srcaddr like /^10\.0\.1\./ and dstaddr like /^10\.0\.2\./
| stats sum(bytes) as total_bytes by srcaddr, dstaddr
| sort total_bytes desc
```

##### Optimization Strategies:
```bash
# Place frequently communicating resources in same AZ
# Use VPC endpoints for AWS services
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.us-east-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids rtb-12345678
```

## 16. Compliance and Governance

### AWS Config Rules for VPC Compliance

#### Step 1: Enable AWS Config
```bash
# Create configuration recorder
aws configservice put-configuration-recorder \
  --configuration-recorder name=default,roleARN=arn:aws:iam::123456789012:role/config-role \
  --recording-group allSupported=true,includeGlobalResourceTypes=true
```

#### Step 2: Create Custom Config Rules
```bash
# Rule to check if security groups allow unrestricted access
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "security-group-ssh-check",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "INCOMING_SSH_DISABLED"
    }
  }'

# Rule to check if VPC flow logging is enabled
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "vpc-flow-logs-enabled",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "VPC_FLOW_LOGS_ENABLED"
    }
  }'
```

### Automated Remediation

#### Step 1: Create Lambda for Auto-Remediation
```python
# auto-remediation-lambda.py
import boto3
import json

def lambda_handler(event, context):
    config = boto3.client('config')
    ec2 = boto3.client('ec2')
    
    # Parse Config rule compliance event
    config_item = event['configurationItem']
    compliance_type = event['newEvaluationResult']['complianceType']
    
    if compliance_type == 'NON_COMPLIANT':
        resource_id = config_item['resourceId']
        
        if config_item['resourceType'] == 'AWS::EC2::SecurityGroup':
            # Remove unrestricted SSH access
            try:
                ec2.revoke_security_group_ingress(
                    GroupId=resource_id,
                    IpPermissions=[
                        {
                            'IpProtocol': 'tcp',
                            'FromPort': 22,
                            'ToPort': 22,
                            'IpRanges': [{'CidrIp': '0.0.0.0/0'}]
                        }
                    ]
                )
                print(f"Removed unrestricted SSH access from {resource_id}")
            except Exception as e:
                print(f"Failed to remediate {resource_id}: {str(e)}")
    
    return {
        'statusCode': 200,
        'body': json.dumps('Remediation completed')
    }
```

## 17. Cost Optimization Strategies

### VPC Cost Analysis

#### Data Transfer Cost Monitoring
```bash
# Create custom metric for data transfer costs
aws cloudwatch put-metric-data \
  --namespace "AWS/Billing" \
  --metric-data MetricName=DataTransferCost,Value=45.67,Unit=None,Dimensions=Name=Currency,Value=USD
```

#### Reserved Instance Planning
```bash
# Analyze instance usage patterns
aws ce get-usage-and-costs \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=INSTANCE_TYPE
```

---

This comprehensive guide covers practical implementation steps for common AWS scenarios. Each section provides both conceptual understanding and hands-on commands for real-world implementation.

## Additional Resources

### AWS CLI Configuration
```bash
# Configure AWS CLI with profiles
aws configure set profile.dev.region us-east-1
aws configure set profile.dev.output json
aws configure set profile.prod.region us-west-2
aws configure set profile.prod.output table

# Use specific profile
aws ec2 describe-instances --profile prod
```

### Useful Aliases and Scripts
```bash
# Add to ~/.bashrc or ~/.zshrc
alias aws-instances='aws ec2 describe-instances --query "Reservations[].Instances[].[InstanceId,State.Name,InstanceType,PublicIpAddress,PrivateIpAddress,Tags[?Key==`Name`].Value|[0]]" --output table'
alias aws-security-groups='aws ec2 describe-security-groups --query "SecurityGroups[].[GroupId,GroupName,Description]" --output table'
alias aws-subnets='aws ec2 describe-subnets --query "Subnets[].[SubnetId,VpcId,CidrBlock,AvailabilityZone,Tags[?Key==`Name`].Value|[0]]" --output table'
```

### Quick Reference Commands
```bash
# Get your current AWS identity
aws sts get-caller-identity

# List all VPCs
aws ec2 describe-vpcs --output table

# Get instance metadata (from within EC2)
curl http://169.254.169.254/latest/meta-data/instance-id
curl http://169.254.169.254/latest/meta-data/local-ipv4

# Check if instance can assume IAM role
aws sts get-caller-identity
```

This guide serves as a comprehensive reference for AWS networking scenarios commonly encountered in interviews and real-world implementations.