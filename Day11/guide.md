# CloudFormation Stack Deployment Guide

## Prerequisites

Before deploying this stack, ensure you have:

1. **AWS CLI configured** with appropriate permissions
2. **Existing EC2 Key Pair** in the target region
3. **IAM permissions** for CloudFormation and all services used
4. **Valid AMI IDs** for your region (update the Mappings section)

## Required IAM Permissions

Your deployment user/role needs these permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudformation:*",
        "ec2:*",
        "elasticloadbalancing:*",
        "autoscaling:*",
        "rds:*",
        "s3:*",
        "iam:*",
        "logs:*",
        "cloudwatch:*",
        "sns:*"
      ],
      "Resource": "*"
    }
  ]
}
```

## Step-by-Step Deployment

### 1. Prepare the Template

Save the CloudFormation template as `application-stack.yaml` and update the AMI IDs in the `AWSRegionArch2AMI` mapping section for your region.

### 2. Validate the Template

```bash
aws cloudformation validate-template --template-body file://application-stack.yaml
```

### 3. Create the Stack

#### Using AWS CLI:

```bash
aws cloudformation create-stack \
  --stack-name myapp-prod-stack \
  --template-body file://application-stack.yaml \
  --parameters \
    ParameterKey=ProjectName,ParameterValue=MyWebApp \
    ParameterKey=Environment,ParameterValue=prod \
    ParameterKey=InstanceType,ParameterValue=t3.small \
    ParameterKey=KeyPairName,ParameterValue=my-keypair \
    ParameterKey=DBUsername,ParameterValue=admin \
    ParameterKey=DBPassword,ParameterValue=SecurePassword123 \
  --capabilities CAPABILITY_NAMED_IAM
```

#### Using Parameter File:

Create `parameters.json`:
```json
[
  {
    "ParameterKey": "ProjectName",
    "ParameterValue": "MyWebApp"
  },
  {
    "ParameterKey": "Environment",
    "ParameterValue": "prod"
  },
  {
    "ParameterKey": "InstanceType",
    "ParameterValue": "t3.small"
  },
  {
    "ParameterKey": "KeyPairName",
    "ParameterValue": "my-keypair"
  },
  {
    "ParameterKey": "DBUsername",
    "ParameterValue": "admin"
  },
  {
    "ParameterKey": "DBPassword",
    "ParameterValue": "SecurePassword123"
  }
]
```

Then deploy:
```bash
aws cloudformation create-stack \
  --stack-name myapp-prod-stack \
  --template-body file://application-stack.yaml \
  --parameters file://parameters.json \
  --capabilities CAPABILITY_NAMED_IAM
```

### 4. Monitor Stack Creation

```bash
# Watch stack events
aws cloudformation describe-stack-events --stack-name myapp-prod-stack

# Check stack status
aws cloudformation describe-stacks --stack-name myapp-prod-stack --query 'Stacks[0].StackStatus'
```

## Stack Outputs

After successful deployment, retrieve important information:

```bash
# Get all outputs
aws cloudformation describe-stacks --stack-name myapp-prod-stack --query 'Stacks[0].Outputs'

# Get specific outputs
aws cloudformation describe-stacks --stack-name myapp-prod-stack --query 'Stacks[0].Outputs[?OutputKey==`LoadBalancerURL`].OutputValue' --output text
```

## Post-Deployment Tasks

### 1. Verify Application

1. Access the Load Balancer URL from the stack outputs
2. Verify the application loads correctly
3. Check health checks in the target group

### 2. Configure Monitoring

1. Review CloudWatch Dashboard
2. Set up SNS notifications for alerts
3. Configure additional alarms as needed

### 3. Database Setup

```bash
# Connect to database via bastion host
ssh -i your-key.pem ec2-user@<bastion-ip>
mysql -h <database-endpoint> -u admin -p

# Create application database and user
CREATE DATABASE myapp;
CREATE USER 'appuser'@'%' IDENTIFIED BY 'app-password';
GRANT ALL PRIVILEGES ON myapp.* TO 'appuser'@'%';
FLUSH PRIVILEGES;
```

### 4. SSL/TLS Configuration

To add HTTPS support:

1. Request ACM certificate
2. Update the Load Balancer listener
3. Add HTTPS security group rules

## Updating the Stack

### Using Change Sets (Recommended)

```bash
# Create change set
aws cloudformation create-change-set \
  --stack-name myapp-prod-stack \
  --template-body file://application-stack-updated.yaml \
  --parameters file://parameters.json \
  --change-set-name update-$(date +%Y%m%d-%H%M%S) \
  --capabilities CAPABILITY_NAMED_IAM

# Review changes
aws cloudformation describe-change-set --change-set-name <change-set-name> --stack-name myapp-prod-stack

# Execute change set
aws cloudformation execute-change-set --change-set-name <change-set-name> --stack-name myapp-prod-stack
```

### Direct Update

```bash
aws cloudformation update-stack \
  --stack-name myapp-prod-stack \
  --template-body file://application-stack-updated.yaml \
  --parameters file://parameters.json \
  --capabilities CAPABILITY_NAMED_IAM
```

## Troubleshooting

### Common Issues

1. **Stack creation fails due to AMI not found**
   - Update AMI IDs in the template mappings

2. **IAM permissions errors**
   - Ensure your user has all required permissions
   - Check if CAPABILITY_NAMED_IAM is specified

3. **Resource limit exceeded**
   - Check AWS service limits for your account
   - Request limit increases if needed

4. **VPC CIDR conflicts**
   - Ensure VPC CIDR doesn't conflict with existing VPCs
   - Modify the VPCCidr parameter

### Debug Commands

```bash
# Get detailed stack information
aws cloudformation describe-stacks --stack-name myapp-prod-stack

# View stack events for troubleshooting
aws cloudformation describe-stack-events --stack-name myapp-prod-stack

# Check resource drift
aws cloudformation detect-stack-drift --stack-name myapp-prod-stack

# Get drift detection results
aws cloudformation describe-stack-resource-drifts --stack-name myapp-prod-stack
```

## Cleanup

### Delete Stack

```bash
# Delete the stack (will remove all resources)
aws cloudformation delete-stack --stack-name myapp-prod-stack

# Monitor deletion progress
aws cloudformation describe-stack-events --stack-name myapp-prod-stack
```

### Manual Cleanup

Some resources may need manual cleanup:

1. **S3 Bucket** - Empty bucket contents first
2. **RDS Snapshots** - Delete manually if not needed
3. **EIP Addresses** - Release if not automatically released

## Best Practices for Production

### 1. Use Nested Stacks

Break down the template into smaller, manageable pieces:
- Network stack (VPC, subnets, gateways)
- Security stack (security groups, IAM roles)
- Application stack (EC2, ALB, ASG)
- Database stack (RDS, backups)

### 2. Parameter Store Integration

Store sensitive parameters in AWS Systems Manager Parameter Store:

```yaml
Parameters:
  DBPassword:
    Type: AWS::SSM::Parameter::Value<String>
    Default: /myapp/prod/db/password
    NoEcho: true
```

### 3. Cross-Stack References

Use exports/imports for sharing resources between stacks:

```yaml
# In network stack
Outputs:
  VPCId:
    Value: !Ref VPC
    Export:
      Name: !Sub '${AWS::StackName}-VPC-ID'

# In application stack
Resources:
  MyInstance:
    Properties:
      VpcId: !ImportValue 'network-stack-VPC-ID'
```

### 4. Environment-Specific Configurations

Use different parameter files for each environment:
- `parameters-dev.json`
- `parameters-staging.json`
- `parameters-prod.json`

### 5. Automated Deployment Pipeline

Integrate with CI/CD pipelines:

```yaml
# GitHub Actions example
- name: Deploy CloudFormation
  run: |
    aws cloudformation deploy \
      --template-file application-stack.yaml \
      --stack-name myapp-${{ env.ENVIRONMENT }}-stack \
      --parameter-overrides file://parameters-${{ env.ENVIRONMENT }}.json \
      --capabilities CAPABILITY_NAMED_IAM \
      --no-fail-on-empty-changeset
```

This comprehensive guide provides everything needed to successfully deploy and manage your CloudFormation application stack.