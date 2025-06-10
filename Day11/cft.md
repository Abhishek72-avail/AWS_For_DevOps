# CloudFormation Hands-On Exercises

## Exercise 1: Basic VPC and EC2 Setup

**Objective:** Create a simple CloudFormation template for a VPC with a single EC2 instance.

**Requirements:**
- VPC with CIDR 10.0.0.0/16
- Public subnet with CIDR 10.0.1.0/24
- Internet Gateway
- EC2 instance in the public subnet
- Security group allowing SSH access

**Template Structure:**
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Exercise 1 - Basic VPC and EC2'

Parameters:
  KeyPairName:
    Type: AWS::EC2::KeyPair::KeyName
    Description: 'EC2 Key Pair for SSH access'

Resources:
  # Add your resources here
  # VPC, Subnet, IGW, RouteTable, SecurityGroup, EC2Instance

Outputs:
  # Add outputs for VPC ID, Instance ID, and Public IP
```

**Tasks:**
1. Complete the template
2. Deploy the stack
3. SSH into the instance using the key pair
4. Verify internet connectivity from the instance

---

## Exercise 2: Multi-AZ Web Application

**Objective:** Extend Exercise 1 to create a multi-AZ setup with load balancing.

**Requirements:**
- 2 public subnets in different AZs
- 2 private subnets in different AZs
- NAT Gateway in each public subnet
- Application Load Balancer
- Auto Scaling Group with instances in private subnets
- Web servers serving a simple HTML page

**New Components to Add:**
- Second availability zone resources
- NAT Gateways
- Application Load Balancer
- Target Group
- Launch Template
- Auto Scaling Group

**Tasks:**
1. Modify your Exercise 1 template
2. Add the new components
3. Deploy and test load balancing
4. Verify instances can reach the internet through NAT Gateways

---

## Exercise 3: Database Integration

**Objective:** Add RDS database to your multi-AZ application.

**Requirements:**
- RDS MySQL database in private subnets
- Database subnet group
- Security group for database access
- Connection from web servers to database
- Bastion host for database administration

**New Components:**
- DB Subnet Group
- RDS Database Instance
- Database Security Group
- Bastion Host

**Tasks:**
1. Add database components to your template
2. Create a simple web application that connects to the database
3. Deploy and verify database connectivity
4. Connect to the database via bastion host

---

## Exercise 4: Monitoring and Alerts

**Objective:** Add comprehensive monitoring to your application stack.

**Requirements:**
- CloudWatch custom metrics
- Alarms for CPU utilization, memory usage, and database connections
- SNS topic for notifications
- CloudWatch dashboard
- Auto scaling based on CloudWatch alarms

**New Components:**
- SNS Topic and Subscription
- CloudWatch Alarms
- Auto Scaling Policies
- CloudWatch Dashboard
- Custom CloudWatch Logs

**Tasks:**
1. Add monitoring components
2. Create custom metrics from your application
3. Test auto scaling by generating load
4. Set up email notifications for alerts

---

## Exercise 5: Security Hardening

**Objective:** Implement security best practices in your CloudFormation template.

**Requirements:**
- IAM roles for EC2 instances
- S3 bucket with encryption
- VPC Flow Logs
- AWS Config rules
- WAF for the Application Load Balancer
- Secrets Manager for database credentials

**Security Enhancements:**
- Replace hardcoded passwords with Secrets Manager
- Add IAM roles with least privilege
- Enable VPC Flow Logs
- Add WAF rules for common attacks
- Enable S3 bucket encryption and versioning

**Tasks:**
1. Implement all security enhancements
2. Test that applications still work
3. Review security group rules for least privilege
4. Verify encryption is working

---

## Exercise 6: Cross-Stack Dependencies

**Objective:** Break your monolithic template into multiple smaller stacks.

**Requirements:**
- Network stack (VPC, subnets, gateways)
- Security stack (security groups, IAM roles)
- Database stack (RDS, backup configuration)
- Application stack (EC2, ALB, ASG)
- Monitoring stack (CloudWatch, SNS)

**Implementation:**
1. Create separate templates for each stack
2. Use Outputs and ImportValue for cross-stack references
3. Deploy stacks in correct order
4. Test updates to individual stacks

**Template Examples:**

**Network Stack (network.yaml):**
```yaml
Outputs:
  VPCId:
    Value: !Ref VPC
    Export:
      Name: !Sub '${AWS::StackName}-VPC-ID'
  
  PublicSubnets:
    Value: !Join [',', [!Ref PublicSubnet1, !Ref PublicSubnet2]]
    Export:
      Name: !Sub '${AWS::StackName}-Public-Subnets'
```

**Application Stack (application.yaml):**
```yaml
Resources:
  ApplicationLoadBalancer:
    Properties:
      Subnets: !Split [',', !ImportValue 'network-stack-Public-Subnets']
```

---

## Exercise 7: CI/CD Integration

**Objective:** Integrate CloudFormation with a CI/CD pipeline.

**Requirements:**
- GitHub Actions or AWS CodePipeline
- Automated testing of templates
- Environment-specific deployments
- Rollback capability
- Slack/email notifications

**Pipeline Steps:**
1. Template validation
2. Security scanning
3. Deploy to development
4. Run integration tests
5. Deploy to staging
6. Manual approval
7. Deploy to production

**GitHub Actions Example:**
```yaml
name: Deploy CloudFormation Stack

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Validate CloudFormation template
        run: |
          aws cloudformation validate-template --template-body file://template.yaml
      
      - name: Deploy to development
        run: |
          aws cloudformation deploy \
            --template-file template.yaml \
            --stack-name myapp-dev-stack \
            --parameter-overrides file://parameters-dev.json \
            --capabilities CAPABILITY_NAMED_IAM
```

---

## Exercise 8: Advanced Patterns

**Objective:** Implement advanced CloudFormation patterns and features.

**Requirements:**
- Custom Resources using Lambda
- Nested stacks
- Stack sets for multi-region deployment
- Drift detection and remediation
- Cost optimization with Spot instances

**Advanced Features:**
1. **Custom Resource for Random Password Generation:**
```yaml
Resources:
  RandomPasswordGenerator:
    Type: AWS::CloudFormation::CustomResource
    Properties:
      ServiceToken: !GetAtt PasswordGeneratorFunction.Arn
      Length: 16
```

2. **Nested Stack for Reusable Components:**
```yaml
Resources:
  NetworkStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/templates/network.yaml
      Parameters:
        VPCCidr: 10.0.0.0/16
```

3. **Conditional Resources:**
```yaml
Conditions:
  CreateSpotInstances: !Equals [!Ref UseSpotInstances, 'true']

Resources:
  SpotLaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Condition: CreateSpotInstances
```

---

## Exercise 9: Cost Optimization

**Objective:** Implement cost optimization strategies in your CloudFormation templates.

**Cost Optimization Techniques:**
1. **Scheduled Auto Scaling:**
```yaml
Resources:
  ScheduledScaleUp:
    Type: AWS::AutoScaling::ScheduledAction
    Properties:
      AutoScalingGroupName: !Ref AutoScalingGroup
      DesiredCapacity: 4
      Recurrence: "0 8 * * MON-FRI"
  
  ScheduledScaleDown:
    Type: AWS::AutoScaling::ScheduledAction
    Properties:
      AutoScalingGroupName: !Ref AutoScalingGroup
      DesiredCapacity: 1
      Recurrence: "0 18 * * MON-FRI"
```

2. **Spot Instances in Auto Scaling:**
```yaml
Resources:
  MixedInstancesLaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateData:
        InstanceMarketOptions:
          MarketType: spot
          SpotOptions:
            MaxPrice: "0.05"
```

3. **RDS Scheduled Start/Stop:**
```yaml
Resources:
  DatabaseStopFunction:
    Type: AWS::Lambda::Function
    Properties:
      Runtime: python3.9
      Handler: index.lambda_handler
      Code:
        ZipFile: |
          import boto3
          def lambda_handler(event, context):
              rds = boto3.client('rds')
              rds.stop_db_instance(DBInstanceIdentifier='mydb')
```

---

## Exercise 10: Disaster Recovery

**Objective:** Implement disaster recovery and backup strategies.

**Requirements:**
- Multi-region deployment
- Automated backups
- Cross-region replication
- Disaster recovery testing
- Recovery time objectives (RTO) and recovery point objectives (RPO)

**DR Components:**
1. **Cross-Region Database Replication:**
```yaml
Resources:
  DatabaseReadReplica:
    Type: AWS::RDS::DBInstance
    Properties:
      SourceDBInstanceIdentifier: !Sub 
        - arn:aws:rds:${SourceRegion}:${AWS::AccountId}:db:${SourceDBInstanceIdentifier}
        - SourceRegion: us-east-1
          SourceDBInstanceIdentifier: !Ref DatabaseInstance
```

2. **S3 Cross-Region Replication:**
```yaml
Resources:
  S3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      ReplicationConfiguration:
        Role: !GetAtt ReplicationRole.Arn
        Rules:
          - Id: ReplicateToSecondaryRegion
            Status: Enabled
            Prefix: data/
            Destination:
              Bucket: !Sub 'arn:aws:s3:::${BackupBucket}'
              StorageClass: STANDARD_IA
```

## Assessment Criteria

For each exercise, evaluate your work based on:

1. **Functionality** - Does the template deploy successfully?
2. **Best Practices** - Are CloudFormation best practices followed?
3. **Security** - Are security best practices implemented?
4. **Documentation** - Is the template well-documented?
5. **Error Handling** - Are errors handled gracefully?
6. **Cost Optimization** - Are resources optimized for cost?
7. **Maintainability** - Is the template easy to maintain and update?

## Bonus Challenges

1. **Create a CloudFormation template that deploys itself** (Bootstrap problem)
2. **Implement blue-green deployment using CloudFormation**
3. **Create a template that can migrate data between versions**
4. **Build a template that automatically scales based on queue depth**
5. **Implement a complete observability stack with metrics, logs, and traces**

## Solutions Repository

Create a GitHub repository with your solutions and document your learning journey. Include:
- Template files
- Parameter files for different environments
- Deployment scripts
- Documentation
- Lessons learned
- Performance metrics

This hands-on approach will give you practical experience with CloudFormation and prepare you for real-world infrastructure automation challenges.