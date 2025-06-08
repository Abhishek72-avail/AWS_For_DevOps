# Day 4: AWS Networking (VPC) - Complete Guide

## Table of Contents
- [Introduction to AWS Networking](#introduction-to-aws-networking)
- [Virtual Private Cloud (VPC) Overview](#virtual-private-cloud-vpc-overview)
- [VPC Components](#vpc-components)
- [Creating and Configuring VPCs](#creating-and-configuring-vpcs)
- [Subnets Deep Dive](#subnets-deep-dive)
- [Route Tables](#route-tables)
- [Internet Gateway and NAT Gateway](#internet-gateway-and-nat-gateway)
- [Security Groups vs NACLs](#security-groups-vs-nacls)
- [VPC Peering](#vpc-peering)
- [VPC Endpoints](#vpc-endpoints)
- [Hands-On Lab Exercises](#hands-on-lab-exercises)
- [Best Practices](#best-practices)
- [Troubleshooting Common Issues](#troubleshooting-common-issues)
- [Additional Resources](#additional-resources)

## Introduction to AWS Networking

AWS networking provides the foundation for building secure, scalable, and highly available cloud infrastructure. Understanding networking concepts is crucial for designing robust applications that can communicate efficiently while maintaining security and performance.

### Key Networking Concepts
- **IP Addressing**: Understanding IPv4 and CIDR notation
- **Subnetting**: Dividing networks into smaller segments
- **Routing**: Directing network traffic between destinations
- **Network Security**: Controlling access and protecting data in transit

## Virtual Private Cloud (VPC) Overview

A VPC is a logically isolated section of the AWS Cloud where you can launch AWS resources in a virtual network that you define. You have complete control over your virtual networking environment.

### VPC Benefits
- **Isolation**: Logically isolated from other virtual networks
- **Control**: Complete control over network configuration
- **Security**: Multiple layers of security controls
- **Flexibility**: Customize network configuration for your needs
- **Integration**: Seamlessly connect to on-premises networks

### VPC Characteristics
- **Region-specific**: Each VPC belongs to a single AWS region
- **CIDR Block**: Must specify an IPv4 CIDR block (e.g., 10.0.0.0/16)
- **Tenancy**: Default or dedicated hardware tenancy
- **DNS**: Built-in DNS resolution and hostname support

## VPC Components

### 1. CIDR Blocks
```
Primary CIDR Block: 10.0.0.0/16 (65,536 IP addresses)
Secondary CIDR Blocks: Additional blocks can be added
IPv6 CIDR Block: Optional IPv6 support
```

### 2. Availability Zones
- VPCs span multiple Availability Zones within a region
- Resources can be distributed across AZs for high availability
- Each AZ is a separate failure domain

### 3. Subnets
- Subdivisions of VPC CIDR block
- Associated with a single Availability Zone
- Can be public or private

### 4. Route Tables
- Control traffic routing within VPC
- Associated with subnets
- Contain routing rules (routes)

### 5. Internet Gateway (IGW)
- Enables internet access for VPC resources
- Horizontally scaled and highly available
- One IGW per VPC

### 6. NAT Gateway/Instance
- Enables outbound internet access for private subnet resources
- Prevents inbound internet access
- Managed service (NAT Gateway) or self-managed (NAT Instance)

## Creating and Configuring VPCs

### Step 1: Planning Your VPC
Before creating a VPC, plan your network architecture:

```
Example VPC Design:
- VPC CIDR: 10.0.0.0/16
- Public Subnet 1: 10.0.1.0/24 (AZ-1a)
- Public Subnet 2: 10.0.2.0/24 (AZ-1b)
- Private Subnet 1: 10.0.3.0/24 (AZ-1a)
- Private Subnet 2: 10.0.4.0/24 (AZ-1b)
```

### Step 2: Creating a VPC via Console
1. Navigate to VPC Dashboard
2. Click "Create VPC"
3. Choose "VPC and more" for guided setup
4. Configure:
   - Name tag: `MyVPC`
   - IPv4 CIDR block: `10.0.0.0/16`
   - IPv6 CIDR block: No IPv6 CIDR block
   - Tenancy: Default
   - Number of Availability Zones: 2
   - Number of public subnets: 2
   - Number of private subnets: 2
   - NAT gateways: 1 per AZ
   - VPC endpoints: S3 Gateway

### Step 3: Creating a VPC via CLI
```bash
# Create VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=MyVPC}]'

# Enable DNS hostnames
aws ec2 modify-vpc-attribute --vpc-id vpc-12345678 --enable-dns-hostnames

# Enable DNS resolution
aws ec2 modify-vpc-attribute --vpc-id vpc-12345678 --enable-dns-support
```

### Step 4: Creating a VPC via CloudFormation
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: MyVPC
```

## Subnets Deep Dive

### Subnet Types

#### Public Subnets
- Have a route to Internet Gateway
- Resources can have public IP addresses
- Suitable for web servers, load balancers, bastion hosts

#### Private Subnets
- No direct route to Internet Gateway
- Resources use NAT Gateway/Instance for outbound internet access
- Suitable for application servers, databases

#### Database Subnets
- Specialized private subnets for databases
- Often in separate subnet groups
- Additional security controls

### Creating Subnets

#### Via Console
1. Navigate to VPC → Subnets
2. Click "Create subnet"
3. Configure:
   - VPC ID: Select your VPC
   - Subnet name: `PublicSubnet1`
   - Availability Zone: `us-east-1a`
   - IPv4 CIDR block: `10.0.1.0/24`

#### Via CLI
```bash
# Create public subnet
aws ec2 create-subnet \
  --vpc-id vpc-12345678 \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PublicSubnet1}]'

# Create private subnet
aws ec2 create-subnet \
  --vpc-id vpc-12345678 \
  --cidr-block 10.0.3.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PrivateSubnet1}]'
```

### Subnet Configuration Best Practices
- Use different AZs for high availability
- Reserve IP addresses for future growth
- Use consistent naming conventions
- Plan CIDR blocks to avoid conflicts

## Route Tables

Route tables determine where network traffic is directed. Each subnet must be associated with a route table.

### Default Route Table
- Automatically created with VPC
- Contains local route for VPC CIDR
- Initially associated with all subnets

### Custom Route Tables
- Created for specific routing requirements
- Can be associated with multiple subnets
- Allow granular traffic control

### Creating Route Tables

#### Via Console
1. Navigate to VPC → Route Tables
2. Click "Create route table"
3. Configure:
   - Name: `PublicRouteTable`
   - VPC: Select your VPC
4. Add routes:
   - Destination: `0.0.0.0/0`
   - Target: Internet Gateway

#### Via CLI
```bash
# Create route table
aws ec2 create-route-table \
  --vpc-id vpc-12345678 \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=PublicRouteTable}]'

# Add route to Internet Gateway
aws ec2 create-route \
  --route-table-id rtb-12345678 \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-12345678

# Associate subnet with route table
aws ec2 associate-route-table \
  --subnet-id subnet-12345678 \
  --route-table-id rtb-12345678
```

### Route Priorities
1. Most specific route (longest prefix match)
2. Local routes (highest priority)
3. Static routes
4. Dynamic routes (BGP)

## Internet Gateway and NAT Gateway

### Internet Gateway (IGW)
- Provides internet access to VPC
- Horizontally scaled and redundant
- No bandwidth constraints
- Performs network address translation (NAT)

#### Creating Internet Gateway
```bash
# Create Internet Gateway
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=MyIGW}]'

# Attach to VPC
aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-12345678 \
  --vpc-id vpc-12345678
```

### NAT Gateway
- Enables outbound internet access for private subnets
- Managed service with high availability
- Charged per hour and data processed
- Placed in public subnet

#### Creating NAT Gateway
```bash
# Allocate Elastic IP
aws ec2 allocate-address --domain vpc

# Create NAT Gateway
aws ec2 create-nat-gateway \
  --subnet-id subnet-12345678 \
  --allocation-id eipalloc-12345678 \
  --tag-specifications 'ResourceType=nat-gateway,Tags=[{Key=Name,Value=MyNATGateway}]'
```

### NAT Instance (Alternative)
- EC2 instance acting as NAT device
- More cost-effective for small workloads
- Requires manual management and scaling
- Single point of failure

## Security Groups vs NACLs

### Security Groups
- **Stateful**: Return traffic automatically allowed
- **Instance-level**: Applied to ENI (Elastic Network Interface)
- **Allow rules only**: Cannot create deny rules
- **Evaluation**: All rules evaluated before decision

#### Security Group Rules
```bash
# Create security group
aws ec2 create-security-group \
  --group-name WebServerSG \
  --description "Security group for web servers" \
  --vpc-id vpc-12345678

# Add inbound HTTP rule
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Add inbound SSH rule
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 22 \
  --source-group sg-87654321
```

### Network ACLs (NACLs)
- **Stateless**: Must create rules for both directions
- **Subnet-level**: Applied to entire subnet
- **Allow and deny rules**: Can explicitly deny traffic
- **Evaluation**: Rules processed in order

#### NACL Rules Example
```bash
# Create Network ACL
aws ec2 create-network-acl \
  --vpc-id vpc-12345678 \
  --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=WebNACL}]'

# Add inbound HTTP rule
aws ec2 create-network-acl-entry \
  --network-acl-id acl-12345678 \
  --rule-number 100 \
  --protocol tcp \
  --port-range From=80,To=80 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow

# Add outbound rule for HTTP responses
aws ec2 create-network-acl-entry \
  --network-acl-id acl-12345678 \
  --rule-number 100 \
  --protocol tcp \
  --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow \
  --egress
```

## VPC Peering

VPC Peering enables private connectivity between VPCs across regions or accounts.

### VPC Peering Characteristics
- **Non-transitive**: Must create direct connections
- **No overlapping CIDR blocks**: IP ranges cannot overlap
- **Cross-region**: Can peer VPCs in different regions
- **Cross-account**: Can peer VPCs in different accounts

### Creating VPC Peering Connection
```bash
# Create peering connection
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-12345678 \
  --peer-vpc-id vpc-87654321 \
  --peer-region us-west-2

# Accept peering connection (in peer region)
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id pcx-12345678

# Add routes for peering
aws ec2 create-route \
  --route-table-id rtb-12345678 \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id pcx-12345678
```

## VPC Endpoints

VPC Endpoints enable private connectivity to AWS services without internet access.

### Gateway Endpoints
- **S3 and DynamoDB only**
- **No additional charges**
- **Route table entries**

#### Creating Gateway Endpoint
```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.us-east-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids rtb-12345678
```

### Interface Endpoints
- **Most AWS services**
- **Charged per hour and data processed**
- **Uses ENI with private IP**

#### Creating Interface Endpoint
```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.us-east-1.ec2 \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-12345678 \
  --security-group-ids sg-12345678
```

## Hands-On Lab Exercises

### Lab 1: Create a Basic VPC with Public and Private Subnets

#### Objective
Create a VPC with public and private subnets, configure routing, and launch instances.

#### Steps
1. Create VPC with CIDR 10.0.0.0/16
2. Create public subnet (10.0.1.0/24) and private subnet (10.0.2.0/24)
3. Create and attach Internet Gateway
4. Create NAT Gateway in public subnet
5. Configure route tables
6. Create security groups
7. Launch EC2 instances in both subnets
8. Test connectivity

#### Expected Outcome
- EC2 in public subnet accessible from internet
- EC2 in private subnet can access internet but not accessible from internet
- Private EC2 can communicate with public EC2

### Lab 2: Implement VPC Peering

#### Objective
Connect two VPCs using VPC peering and enable cross-VPC communication.

#### Steps
1. Create two VPCs with non-overlapping CIDR blocks
2. Create subnets in each VPC
3. Create VPC peering connection
4. Update route tables
5. Configure security groups
6. Test connectivity between VPCs

### Lab 3: Configure VPC Endpoints

#### Objective
Set up VPC endpoints for S3 and EC2 services to enable private access.

#### Steps
1. Create VPC with private subnets
2. Create S3 Gateway endpoint
3. Create EC2 Interface endpoint
4. Test private access to AWS services
5. Monitor VPC endpoint usage

## Best Practices

### Network Design
- **Plan CIDR blocks carefully** to avoid future conflicts
- **Use multiple AZs** for high availability
- **Separate public and private subnets** based on security requirements
- **Reserve IP space** for future growth
- **Use consistent naming conventions** for resources

### Security
- **Apply principle of least privilege** in security groups
- **Use NACLs for additional subnet-level security**
- **Regularly review and audit** security group rules
- **Enable VPC Flow Logs** for monitoring and troubleshooting
- **Use VPC endpoints** to keep traffic within AWS network

### Performance
- **Place resources in same AZ** when low latency is required
- **Use placement groups** for high-performance computing
- **Consider enhanced networking** for high bandwidth applications
- **Monitor network performance** with CloudWatch metrics

### Cost Optimization
- **Use NAT Gateway instead of NAT Instance** for managed service benefits
- **Right-size NAT Gateways** based on bandwidth requirements
- **Use VPC endpoints** to reduce NAT Gateway costs
- **Review and optimize** data transfer costs

## Troubleshooting Common Issues

### Connectivity Issues

#### Instance Cannot Access Internet
**Symptoms**: Private instance cannot reach internet
**Troubleshooting Steps**:
1. Verify NAT Gateway is running and healthy
2. Check route table has route to NAT Gateway
3. Confirm security groups allow outbound traffic
4. Verify NACLs allow bidirectional traffic

#### Cannot SSH to Instance
**Symptoms**: Cannot connect to EC2 instance via SSH
**Troubleshooting Steps**:
1. Verify instance has public IP (for public subnet)
2. Check security group allows SSH (port 22)
3. Confirm source IP is allowed
4. Verify key pair is correct
5. Check NACL rules

#### VPC Peering Not Working
**Symptoms**: Cannot communicate between peered VPCs
**Troubleshooting Steps**:
1. Verify peering connection is active
2. Check route tables have peering routes
3. Confirm security groups allow cross-VPC traffic
4. Verify no overlapping CIDR blocks

### DNS Resolution Issues
**Symptoms**: Cannot resolve hostnames within VPC
**Troubleshooting Steps**:
1. Enable DNS resolution in VPC settings
2. Enable DNS hostnames in VPC settings
3. Check security groups allow DNS traffic (port 53)
4. Verify instances are using VPC DNS server

### Performance Issues
**Symptoms**: Slow network performance
**Troubleshooting Steps**:
1. Check instance types support enhanced networking
2. Verify instances are in same AZ if low latency needed
3. Monitor network utilization with CloudWatch
4. Consider using placement groups
5. Check for bandwidth limitations

## Monitoring and Logging

### VPC Flow Logs
```bash
# Create VPC Flow Logs
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-12345678 \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name VPCFlowLogs
```

### CloudWatch Metrics
- Monitor NAT Gateway bandwidth and connection count
- Track VPC endpoint usage and performance
- Set up alarms for unusual network activity

### Cost Monitoring
- Monitor data transfer costs
- Track NAT Gateway and VPC endpoint usage
- Set up billing alerts for network-related charges

## Advanced Topics

### Transit Gateway
- Centralized connectivity hub for multiple VPCs
- Simplifies network architecture
- Supports cross-region peering

### AWS PrivateLink
- Securely access services over private connection
- No need for internet gateway or NAT
- Keeps traffic within AWS network

### Direct Connect
- Dedicated network connection to AWS
- Consistent network performance
- Reduced data transfer costs

### Site-to-Site VPN
- Encrypted connection between on-premises and AWS
- Redundant tunnels for high availability
- Route-based or policy-based routing

## Additional Resources

### AWS Documentation
- [Amazon VPC User Guide](https://docs.aws.amazon.com/vpc/)
- [VPC Networking Examples](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Scenarios.html)
- [VPC Security Best Practices](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-best-practices.html)

### AWS Training and Resources
- AWS VPC Fundamentals Course
- AWS Networking Specialty Certification
- AWS Well-Architected Framework - Reliability Pillar

### Tools and Utilities
- AWS VPC Designer
- AWS Config for compliance monitoring
- AWS Trusted Advisor for optimization recommendations
- Third-party network monitoring tools

### Community Resources
- AWS VPC GitHub examples
- AWS Architecture Center
- AWS re:Invent networking sessions
- AWS Community forums and discussion groups

---

## Summary

This comprehensive guide covers AWS VPC networking fundamentals, including creating and configuring VPCs, subnets, and route tables. You've learned about security groups, NACLs, VPC peering, and endpoints. The hands-on labs provide practical experience, while best practices ensure optimal security, performance, and cost efficiency. Continue practicing these concepts and explore advanced networking features as you build more complex AWS architectures.