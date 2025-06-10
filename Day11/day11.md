# AWS CloudFormation Complete Guide - Day 11

## What is AWS CloudFormation?

AWS CloudFormation is a service that allows you to model and provision AWS resources using templates written in JSON or YAML. It enables Infrastructure as Code (IaC), making your infrastructure reproducible, version-controlled, and automated.

## Key Concepts

### 1. Templates
- **JSON or YAML files** that describe AWS resources
- **Declarative** - you specify what you want, not how to create it
- **Version controlled** and **reusable**

### 2. Stacks
- **Collection of AWS resources** managed as a single unit
- Resources are created, updated, or deleted together
- Each stack has a unique name within a region

### 3. Change Sets
- **Preview changes** before applying them
- Shows what will be added, modified, or deleted
- Helps prevent unintended changes

## CloudFormation Template Structure

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Template description'

Parameters:
  # Input parameters for customization

Mappings:
  # Static lookup tables

Conditions:
  # Conditional logic

Resources:
  # AWS resources to create (REQUIRED)

Outputs:
  # Values to return after stack creation
```

## Template Sections Explained

### Parameters
Allow customization of templates:
```yaml
Parameters:
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues: [t3.micro, t3.small, t3.medium]
    Description: EC2 instance type
```

### Mappings
Static lookup tables:
```yaml
Mappings:
  RegionMap:
    us-east-1:
      AMI: ami-0abcdef1234567890
    us-west-2:
      AMI: ami-0fedcba0987654321
```

### Conditions
Conditional resource creation:
```yaml
Conditions:
  CreateProdResources: !Equals [!Ref Environment, 'production']
```

### Resources
The core section defining AWS resources:
```yaml
Resources:
  MyEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !FindInMap [RegionMap, !Ref 'AWS::Region', AMI]
      InstanceType: !Ref InstanceType
```

### Outputs
Return values from the stack:
```yaml
Outputs:
  InstanceId:
    Description: EC2 Instance ID
    Value: !Ref MyEC2Instance
    Export:
      Name: !Sub '${AWS::StackName}-InstanceId'
```

## Intrinsic Functions

CloudFormation provides built-in functions:

- `!Ref` - References parameters or resources
- `!GetAtt` - Gets attribute values from resources
- `!Sub` - Substitutes variables in strings
- `!Join` - Joins values with a delimiter
- `!FindInMap` - Finds values in mappings
- `!If` - Conditional values
- `!Base64` - Encodes strings to Base64

## Best Practices

### 1. Template Organization
- Use descriptive resource names
- Add comprehensive descriptions
- Group related resources logically
- Use consistent naming conventions

### 2. Parameterization
- Make templates reusable with parameters
- Provide sensible defaults
- Use parameter constraints for validation
- Document parameter purposes

### 3. Security
- Use IAM roles instead of hardcoded credentials
- Apply least privilege principle
- Use AWS Secrets Manager for sensitive data
- Enable encryption where possible

### 4. Error Handling
- Use rollback configuration
- Implement proper error handling
- Test templates in development first
- Use change sets for production updates

## Common Resource Types

### Networking
- `AWS::EC2::VPC` - Virtual Private Cloud
- `AWS::EC2::Subnet` - Subnets
- `AWS::EC2::InternetGateway` - Internet Gateway
- `AWS::EC2::RouteTable` - Route Tables
- `AWS::EC2::SecurityGroup` - Security Groups

### Compute
- `AWS::EC2::Instance` - EC2 Instances
- `AWS::AutoScaling::AutoScalingGroup` - Auto Scaling Groups
- `AWS::ElasticLoadBalancingV2::LoadBalancer` - Application Load Balancers

### Storage
- `AWS::S3::Bucket` - S3 Buckets
- `AWS::EBS::Volume` - EBS Volumes

## Working with Stacks

### Creating a Stack
```bash
# AWS CLI
aws cloudformation create-stack \
  --stack-name my-app-stack \
  --template-body file://template.yaml \
  --parameters ParameterKey=InstanceType,ParameterValue=t3.small

# AWS Console
# Upload template file and configure parameters
```

### Updating a Stack
```bash
# Create change set first
aws cloudformation create-change-set \
  --stack-name my-app-stack \
  --template-body file://updated-template.yaml \
  --change-set-name my-changes

# Execute change set
aws cloudformation execute-change-set \
  --change-set-name my-changes \
  --stack-name my-app-stack
```

### Deleting a Stack
```bash
aws cloudformation delete-stack --stack-name my-app-stack
```

## Stack Dependencies

### Cross-Stack References
Export values from one stack and import in another:

```yaml
# In Stack A
Outputs:
  VPCId:
    Value: !Ref MyVPC
    Export:
      Name: !Sub '${AWS::StackName}-VPC-ID'

# In Stack B
Resources:
  MyInstance:
    Type: AWS::EC2::Instance
    Properties:
      SubnetId: !ImportValue 'StackA-VPC-ID'
```

### Nested Stacks
Use child templates within parent templates:

```yaml
Resources:
  NetworkStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-bucket/network-template.yaml
      Parameters:
        VPCCidr: 10.0.0.0/16
```

## Troubleshooting

### Common Issues
1. **Resource naming conflicts** - Use unique names
2. **IAM permissions** - Ensure proper permissions
3. **Resource limits** - Check AWS service limits
4. **Circular dependencies** - Review resource dependencies

### Debugging Tools
- CloudFormation Events tab
- AWS CloudTrail logs
- CloudWatch Logs
- Stack rollback information

## Monitoring and Maintenance

### Stack Drift Detection
Detect when resources have been modified outside CloudFormation:

```bash
aws cloudformation detect-stack-drift --stack-name my-app-stack
```

### Stack Policies
Protect critical resources from updates:

```json
{
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "Update:*",
      "Resource": "LogicalResourceId/ProductionDatabase"
    }
  ]
}
```

## Integration with CI/CD

### Automated Deployments
```yaml
# GitHub Actions example
- name: Deploy CloudFormation Stack
  run: |
    aws cloudformation deploy \
      --template-file template.yaml \
      --stack-name ${{ env.STACK_NAME }} \
      --parameter-overrides InstanceType=t3.small \
      --capabilities CAPABILITY_IAM
```

This comprehensive guide provides the foundation for understanding and working with AWS CloudFormation. The project template that follows will demonstrate these concepts in practice.