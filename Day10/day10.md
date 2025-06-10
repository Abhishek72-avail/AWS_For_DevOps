# AWS CLI Guide for DevOps Engineers

## Table of Contents
- [Introduction](#introduction)
- [Why AWS CLI is Important](#why-aws-cli-is-important)
- [Where AWS CLI is Used](#where-aws-cli-is-used)
- [Installation](#installation)
- [Essential Commands](#essential-commands)
  - [Configuration and Authentication](#configuration-and-authentication)
  - [EC2 (Compute)](#ec2-compute)
  - [S3 (Storage)](#s3-storage)
  - [IAM (Identity and Access Management)](#iam-identity-and-access-management)
  - [Lambda (Serverless)](#lambda-serverless)
  - [RDS (Database)](#rds-database)
  - [CloudFormation (Infrastructure as Code)](#cloudformation-infrastructure-as-code)
  - [CloudWatch (Monitoring)](#cloudwatch-monitoring)
  - [ECS (Container Service)](#ecs-container-service)
  - [Route 53 (DNS)](#route-53-dns)
  - [VPC (Networking)](#vpc-networking)
- [Useful CLI Tips](#useful-cli-tips)
- [Best Practices](#best-practices)

## Introduction

AWS CLI (Command Line Interface) is a crucial tool for DevOps engineers that allows you to interact with AWS services directly from the command line. This guide covers essential commands and use cases for effective AWS resource management.

## Why AWS CLI is Important

AWS CLI is essential for DevOps because it enables:

- **Infrastructure as Code (IaC)**: Programmatically manage AWS resources
- **Automated Deployments**: Script deployment processes
- **CI/CD Pipeline Integration**: Seamlessly integrate with build and deployment pipelines
- **Batch Operations**: Perform bulk operations efficiently
- **Cost-Effective Management**: Avoid GUI overhead for repetitive tasks

## Where AWS CLI is Used

- **CI/CD Pipelines**: Automating deployments, updating services, and managing infrastructure
- **Infrastructure Automation**: Creating and managing resources through scripts
- **Backup and Monitoring**: Automated backup scripts and resource monitoring
- **Development Workflows**: Quick resource provisioning and testing environments
- **System Administration**: Bulk operations and resource management

## Installation

```bash
# Install AWS CLI v2 on Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Install on macOS
brew install awscli

# Install on Windows
# Download and run the MSI installer from AWS documentation

# Verify installation
aws --version
```

## Essential Commands

### Configuration and Authentication

```bash
# Initial setup - configure credentials and default region
aws configure

# View current configuration
aws configure list

# Verify credentials and identity
aws sts get-caller-identity

# Set default region
aws configure set region us-west-2

# Configure named profiles
aws configure --profile production
```

### EC2 (Compute)

```bash
# List all instances
aws ec2 describe-instances

# Launch a new instance
aws ec2 run-instances --image-id ami-12345 --instance-type t2.micro --key-name MyKeyPair

# Start instances
aws ec2 start-instances --instance-ids i-1234567890abcdef0

# Stop instances
aws ec2 stop-instances --instance-ids i-1234567890abcdef0

# Terminate instances
aws ec2 terminate-instances --instance-ids i-1234567890abcdef0

# List security groups
aws ec2 describe-security-groups

# Create key pair
aws ec2 create-key-pair --key-name MyKeyPair --output text --query 'KeyMaterial' > MyKeyPair.pem

# Describe instance status
aws ec2 describe-instance-status --instance-ids i-1234567890abcdef0
```

### S3 (Storage)

```bash
# List all buckets
aws s3 ls

# List objects in a bucket
aws s3 ls s3://bucket-name

# List objects with details
aws s3 ls s3://bucket-name --recursive --human-readable --summarize

# Upload file to bucket
aws s3 cp file.txt s3://bucket-name/

# Download file from bucket
aws s3 cp s3://bucket-name/file.txt .

# Sync local folder with S3 bucket
aws s3 sync ./folder s3://bucket-name/

# Create new bucket
aws s3 mb s3://new-bucket-name

# Delete bucket (with all contents)
aws s3 rb s3://bucket-name --force

# Move files
aws s3 mv s3://bucket-name/old-file.txt s3://bucket-name/new-file.txt
```

### IAM (Identity and Access Management)

```bash
# List all users
aws iam list-users

# List all roles
aws iam list-roles

# Create new user
aws iam create-user --user-name newuser

# Attach policy to user
aws iam attach-user-policy --user-name newuser --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

# List attached user policies
aws iam list-attached-user-policies --user-name username

# Create access key for user
aws iam create-access-key --user-name newuser

# List groups
aws iam list-groups

# Create role
aws iam create-role --role-name MyRole --assume-role-policy-document file://trust-policy.json
```

### Lambda (Serverless)

```bash
# List all functions
aws lambda list-functions

# Invoke function
aws lambda invoke --function-name myFunction response.json

# Update function code
aws lambda update-function-code --function-name myFunction --zip-file fileb://function.zip

# Create function
aws lambda create-function --function-name myFunction --runtime python3.9 --role arn:aws:iam::123456789012:role/execution_role --handler lambda_function.lambda_handler --zip-file fileb://function.zip

# Get function configuration
aws lambda get-function-configuration --function-name myFunction

# Delete function
aws lambda delete-function --function-name myFunction
```

### RDS (Database)

```bash
# List database instances
aws rds describe-db-instances

# Create database instance
aws rds create-db-instance --db-instance-identifier mydb --db-instance-class db.t3.micro --engine mysql --master-username admin --master-user-password mypassword --allocated-storage 20

# Start database instance
aws rds start-db-instance --db-instance-identifier mydb

# Stop database instance
aws rds stop-db-instance --db-instance-identifier mydb

# Create database snapshot
aws rds create-db-snapshot --db-instance-identifier mydb --db-snapshot-identifier mydb-snapshot

# List database snapshots
aws rds describe-db-snapshots --db-instance-identifier mydb
```

### CloudFormation (Infrastructure as Code)

```bash
# List all stacks
aws cloudformation list-stacks

# Create stack
aws cloudformation create-stack --stack-name mystack --template-body file://template.yaml

# Update stack
aws cloudformation update-stack --stack-name mystack --template-body file://template.yaml

# Delete stack
aws cloudformation delete-stack --stack-name mystack

# Describe stack details
aws cloudformation describe-stacks --stack-name mystack

# Validate template
aws cloudformation validate-template --template-body file://template.yaml

# List stack events
aws cloudformation describe-stack-events --stack-name mystack
```

### CloudWatch (Monitoring)

```bash
# List available metrics
aws cloudwatch list-metrics

# Get metric statistics
aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUUtilization --dimensions Name=InstanceId,Value=i-1234567890abcdef0 --statistics Average --start-time 2023-01-01T00:00:00Z --end-time 2023-01-01T23:59:59Z --period 3600

# List log groups
aws logs describe-log-groups

# Tail log group (follow new entries)
aws logs tail /aws/lambda/function-name --follow

# Create log group
aws logs create-log-group --log-group-name my-log-group

# Put log events
aws logs put-log-events --log-group-name my-log-group --log-stream-name my-stream
```

### ECS (Container Service)

```bash
# List clusters
aws ecs list-clusters

# List services in cluster
aws ecs list-services --cluster mycluster

# Update service (scale up/down)
aws ecs update-service --cluster mycluster --service myservice --desired-count 3

# Describe tasks
aws ecs describe-tasks --cluster mycluster --tasks task-id

# List task definitions
aws ecs list-task-definitions

# Run task
aws ecs run-task --cluster mycluster --task-definition mytask:1
```

### Route 53 (DNS)

```bash
# List hosted zones
aws route53 list-hosted-zones

# List resource record sets
aws route53 list-resource-record-sets --hosted-zone-id Z123456789

# Create hosted zone
aws route53 create-hosted-zone --name example.com --caller-reference unique-string

# Change resource record sets
aws route53 change-resource-record-sets --hosted-zone-id Z123456789 --change-batch file://change-batch.json
```

### VPC (Networking)

```bash
# List VPCs
aws ec2 describe-vpcs

# List subnets
aws ec2 describe-subnets

# List route tables
aws ec2 describe-route-tables

# Create VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16

# Create subnet
aws ec2 create-subnet --vpc-id vpc-12345678 --cidr-block 10.0.1.0/24

# Create internet gateway
aws ec2 create-internet-gateway

# Describe network ACLs
aws ec2 describe-network-acls
```

## Useful CLI Tips

### Output Formats
```bash
# Different output formats
aws ec2 describe-instances --output table
aws ec2 describe-instances --output json
aws ec2 describe-instances --output text
```

### Filtering with JMESPath
```bash
# Filter specific fields
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,State.Name]'

# Filter by tags
aws ec2 describe-instances --query 'Reservations[*].Instances[?Tags[?Key==`Environment` && Value==`Production`]]'
```

### Using Profiles
```bash
# Configure multiple profiles
aws configure --profile production
aws configure --profile development

# Use specific profile
aws s3 ls --profile production
```

### Dry Run Commands
```bash
# Test commands without executing
aws ec2 run-instances --dry-run --image-id ami-12345 --instance-type t2.micro
```

### Pagination
```bash
# Handle large result sets
aws s3api list-objects-v2 --bucket mybucket --max-items 1000
```

## Best Practices

1. **Security**
   - Use IAM roles instead of access keys when possible
   - Follow principle of least privilege
   - Regularly rotate access keys
   - Never hardcode credentials in scripts

2. **Automation**
   - Use scripts for repetitive tasks
   - Implement error handling in scripts
   - Use CloudFormation for infrastructure management
   - Version control your scripts

3. **Monitoring**
   - Enable CloudTrail for API logging
   - Use CloudWatch for monitoring
   - Set up billing alerts
   - Regular backup and disaster recovery testing

4. **Organization**
   - Use consistent naming conventions
   - Tag all resources appropriately
   - Use multiple AWS accounts for different environments
   - Document your infrastructure and processes

---

## Contributing

Feel free to submit issues and enhancement requests!

## License

This guide is provided as-is for educational purposes.

---

**Happy DevOps-ing! 🚀**