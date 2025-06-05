# AWS EC2 Complete Guide - Day 3 AWS for DevOps

## Table of Contents
1. [What is AWS EC2?](#what-is-aws-ec2)
2. [Key Features](#key-features)
3. [Real-world Use Cases](#real-world-use-cases)
4. [EC2 Instance Types](#ec2-instance-types)
5. [Ways to Connect to EC2](#ways-to-connect-to-ec2)
6. [Step-by-Step EC2 Setup](#step-by-step-ec2-setup)
7. [Hands-on Projects](#hands-on-projects)
8. [Important Concepts](#important-concepts)
9. [Best Practices](#best-practices)
10. [Troubleshooting](#troubleshooting)

---

## What is AWS EC2?

**Amazon Elastic Compute Cloud (EC2)** is a web service that provides resizable compute capacity in the cloud. It's designed to make web-scale cloud computing easier for developers.

### Core Concepts:
- **Virtual Servers**: EC2 provides virtual machines (instances) in the cloud
- **Scalable**: Can scale up or down based on demand
- **Pay-as-you-use**: Only pay for the compute time you use
- **Complete Control**: Full administrative access to your instances

---

## Key Features

### 1. **Elasticity**
- Auto-scaling based on demand
- Launch or terminate instances as needed

### 2. **Instance Types**
- Various configurations for different workloads
- CPU, memory, storage, and networking optimized options

### 3. **Security**
- Security Groups (virtual firewalls)
- Key Pairs for secure access
- VPC integration

### 4. **Storage Options**
- EBS (Elastic Block Store)
- Instance Store
- EFS (Elastic File System)

### 5. **Load Balancing**
- Application Load Balancer (ALB)
- Network Load Balancer (NLB)
- Classic Load Balancer

---

## Real-world Use Cases

### 1. **Web Applications**
- Hosting websites and web applications
- E-commerce platforms
- Content management systems

### 2. **Development & Testing**
- Development environments
- Staging servers
- CI/CD pipelines

### 3. **Big Data & Analytics**
- Data processing workloads
- Machine learning training
- Analytics platforms

### 4. **Enterprise Applications**
- ERP systems
- CRM applications
- Database servers

### 5. **Backup & Disaster Recovery**
- Backup solutions
- Disaster recovery sites
- Data archival

### 6. **Gaming**
- Game servers
- Multiplayer gaming backends
- Real-time processing

---

## EC2 Instance Types

### General Purpose
- **t3/t4g**: Burstable performance (web servers, small databases)
- **m5/m6i**: Balanced compute, memory, networking

### Compute Optimized
- **c5/c6i**: High-performance processors (HPC, gaming, scientific computing)

### Memory Optimized
- **r5/r6i**: Fast performance for memory-intensive applications
- **x1e**: High memory for Apache Spark, databases

### Storage Optimized
- **i3**: High sequential read/write (NoSQL databases)
- **d3**: Distributed file systems

### Accelerated Computing
- **p3/p4**: GPU instances for ML, HPC
- **g4**: GPU for graphics-intensive applications

---

## Ways to Connect to EC2

### 1. **SSH (Linux Instances)**
```bash
ssh -i "your-key.pem" ec2-user@your-instance-public-ip
```

### 2. **RDP (Windows Instances)**
- Use Remote Desktop Connection
- Default username: Administrator

### 3. **AWS Systems Manager Session Manager**
- Browser-based shell access
- No need for SSH keys or open ports

### 4. **EC2 Instance Connect**
- Browser-based SSH connection
- Temporary SSH keys

### 5. **AWS CLI**
```bash
aws ec2-instance-connect send-ssh-public-key \
    --instance-id i-1234567890abcdef0 \
    --instance-os-user ec2-user \
    --ssh-public-key file://my_rsa_key.pub
```

---

## Step-by-Step EC2 Setup

### Prerequisites
- AWS Account
- Basic understanding of Linux/Windows
- AWS CLI installed (optional)

### Step 1: Launch EC2 Instance

1. **Login to AWS Console**
   - Navigate to EC2 Dashboard
   - Click "Launch Instance"

2. **Choose AMI (Amazon Machine Image)**
   ```
   - Amazon Linux 2 (Free Tier eligible)
   - Ubuntu Server 20.04 LTS
   - Windows Server 2019
   - Red Hat Enterprise Linux
   ```

3. **Choose Instance Type**
   ```
   - t2.micro (Free Tier eligible)
   - t3.small (For production workloads)
   - m5.large (Balanced performance)
   ```

4. **Configure Instance Details**
   - Number of instances: 1
   - Network: Default VPC
   - Subnet: Public subnet
   - Auto-assign Public IP: Enable

5. **Add Storage**
   - Root Volume: 8 GB (Free Tier)
   - Volume Type: gp2 (General Purpose SSD)

6. **Add Tags**
   ```
   Key: Name, Value: MyFirstEC2Instance
   Key: Environment, Value: Development
   Key: Project, Value: DevOps-Learning
   ```

7. **Configure Security Group**
   ```
   Type: SSH, Protocol: TCP, Port: 22, Source: My IP
   Type: HTTP, Protocol: TCP, Port: 80, Source: Anywhere
   Type: HTTPS, Protocol: TCP, Port: 443, Source: Anywhere
   ```

8. **Review and Launch**
   - Create new key pair or use existing
   - Download .pem file (keep it secure!)

### Step 2: Connect to Your Instance

#### For Linux:
```bash
# Set permissions for key file
chmod 400 your-key.pem

# Connect via SSH
ssh -i "your-key.pem" ec2-user@your-public-ip
```

#### For Windows:
```bash
# Get password using key pair
aws ec2 get-password-data --instance-id i-1234567890abcdef0 --priv-launch-key your-key.pem

# Use RDP with Administrator username and decrypted password
```

### Step 3: Basic Server Configuration

```bash
# Update system packages
sudo yum update -y  # Amazon Linux
sudo apt update && sudo apt upgrade -y  # Ubuntu

# Install essential packages
sudo yum install -y git wget curl htop  # Amazon Linux
sudo apt install -y git wget curl htop  # Ubuntu

# Install Docker (for DevOps)
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
sudo usermod -aG docker $USER

# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

---

## Hands-on Projects

### Project 1: Simple Web Server
```bash
# Install Apache/Nginx
sudo yum install -y httpd  # Amazon Linux
sudo systemctl start httpd
sudo systemctl enable httpd

# Create simple HTML page
echo "<h1>Welcome to AWS EC2!</h1>" | sudo tee /var/www/html/index.html

# Access via browser: http://your-public-ip
```

### Project 2: Docker Container Deployment
```bash
# Run Nginx container
docker run -d -p 80:80 --name my-nginx nginx

# Create custom HTML
docker exec -it my-nginx bash
echo "<h1>Dockerized App on EC2</h1>" > /usr/share/nginx/html/index.html
exit
```

### Project 3: CI/CD Pipeline Setup
```bash
# Install Jenkins
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
sudo yum install -y jenkins java-11-openjdk
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Access Jenkins: http://your-public-ip:8080
```

### Project 4: Load Balancer Setup
1. Launch 2 EC2 instances
2. Install web servers on both
3. Create Application Load Balancer
4. Add instances to target group
5. Test load distribution

### Project 5: Auto Scaling Group
1. Create Launch Template
2. Configure Auto Scaling Group
3. Set scaling policies
4. Test scaling scenarios

---

## Important Concepts

### 1. **Security Groups**
- Act as virtual firewalls
- Control inbound and outbound traffic
- Stateful (return traffic automatically allowed)

### 2. **Key Pairs**
- Used for secure access to instances
- Public key stored on instance
- Private key stays with you

### 3. **Elastic IP**
- Static public IP address
- Remains same even after instance restart
- Charged when not associated with running instance

### 4. **AMI (Amazon Machine Image)**
- Template for EC2 instances
- Contains OS, applications, and configurations
- Can create custom AMIs

### 5. **Instance Metadata**
```bash
# Get instance metadata
curl http://169.254.169.254/latest/meta-data/
curl http://169.254.169.254/latest/meta-data/public-ipv4
curl http://169.254.169.254/latest/meta-data/instance-id
```

### 6. **User Data Scripts**
```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Automated Setup Complete!</h1>" > /var/www/html/index.html
```

---

## Best Practices

### 1. **Security**
- Use IAM roles instead of hardcoded credentials
- Regularly update security groups
- Enable CloudTrail for auditing
- Use Systems Manager for patch management

### 2. **Cost Optimization**
- Use appropriate instance types
- Implement auto-scaling
- Use Spot Instances for non-critical workloads
- Regular cleanup of unused resources

### 3. **Monitoring**
- Enable detailed monitoring
- Use CloudWatch alarms
- Implement log aggregation
- Monitor resource utilization

### 4. **Backup & Recovery**
- Regular EBS snapshots
- Multi-AZ deployments
- Implement disaster recovery procedures

### 5. **Tagging Strategy**
```
Environment: Production/Development/Testing
Project: ProjectName
Owner: TeamName
CostCenter: Department
```

---

## Troubleshooting

### Common Issues:

#### 1. **Cannot Connect via SSH**
```bash
# Check security group allows SSH (port 22)
# Verify key pair permissions: chmod 400 key.pem
# Ensure using correct username (ec2-user, ubuntu, admin)
```

#### 2. **Instance Not Accessible**
- Check if public IP is assigned
- Verify route tables and internet gateway
- Check NACL rules

#### 3. **High CPU Usage**
```bash
# Check processes
top
htop

# Check system logs
sudo tail -f /var/log/messages
```

#### 4. **Storage Issues**
```bash
# Check disk usage
df -h

# Clear temporary files
sudo find /tmp -type f -atime +7 -delete
```

#### 5. **Network Connectivity**
```bash
# Test connectivity
ping google.com
nslookup google.com
telnet destination-ip port
```

---

## DevOps Integration

### 1. **Infrastructure as Code**
```yaml
# CloudFormation template example
Resources:
  MyEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-0abcdef1234567890
      InstanceType: t2.micro
      KeyName: my-key-pair
      SecurityGroups:
        - !Ref MySecurityGroup
```

### 2. **Terraform Example**
```hcl
resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t2.micro"
  key_name      = "my-key-pair"
  
  vpc_security_group_ids = [aws_security_group.web.id]
  
  tags = {
    Name = "HelloWorld"
  }
}
```

### 3. **Ansible Playbook**
```yaml
- name: Deploy web application
  hosts: ec2_instances
  become: yes
  tasks:
    - name: Install Apache
      yum:
        name: httpd
        state: present
    
    - name: Start Apache
      service:
        name: httpd
        state: started
        enabled: yes
```

---

## Cost Calculator

### Free Tier Limits:
- 750 hours per month of t2.micro instances
- 30 GB of EBS storage
- 2 million I/O operations

### Pricing Factors:
- Instance type and size
- Running time
- Data transfer
- EBS storage
- Elastic IP (when not in use)

---

## Next Steps

1. **Day 4**: Explore EBS and EFS storage options
2. **Day 5**: Learn about VPC and networking
3. **Day 6**: Implement Auto Scaling and Load Balancing
4. **Day 7**: Setup monitoring with CloudWatch
5. **Day 8**: Practice Infrastructure as Code

---

## Resources

- [AWS EC2 Documentation](https://docs.aws.amazon.com/ec2/)
- [EC2 Instance Types](https://aws.amazon.com/ec2/instance-types/)
- [AWS Free Tier](https://aws.amazon.com/free/)
- [EC2 Pricing Calculator](https://calculator.aws/)

---

## Summary

AWS EC2 is the foundation of cloud computing, providing scalable virtual servers for any workload. Master EC2 to build robust, scalable applications in the cloud. Practice with real projects to gain hands-on experience!

Remember: Start small, learn continuously, and always follow security best practices!

---

*Happy Learning! 🚀*