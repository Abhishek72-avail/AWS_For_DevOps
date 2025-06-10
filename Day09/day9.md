# AWS S3 Complete Guide - Day 9 DevOps Learning

## Table of Contents
1. [What is Amazon S3?](#what-is-amazon-s3)
2. [Key Concepts and Terminology](#key-concepts-and-terminology)
3. [S3 Storage Classes](#s3-storage-classes)
4. [S3 Security Features](#s3-security-features)
5. [S3 Versioning and Lifecycle](#s3-versioning-and-lifecycle)
6. [Creating S3 Buckets with Infrastructure as Code](#creating-s3-buckets-with-infrastructure-as-code)
7. [Real-World Examples](#real-world-examples)
8. [Best Practices](#best-practices)
9. [Monitoring and Logging](#monitoring-and-logging)
10. [Cost Optimization](#cost-optimization)

## What is Amazon S3?

Amazon Simple Storage Service (S3) is an object storage service that offers industry-leading scalability, data availability, security, and performance. S3 is designed to store and retrieve any amount of data from anywhere on the web.

### Key Features
- **Durability**: 99.999999999% (11 9's) durability
- **Availability**: 99.99% availability SLA
- **Scalability**: Virtually unlimited storage
- **Security**: Multiple encryption options and access controls
- **Performance**: High throughput and low latency
- **Cost-effective**: Pay only for what you use

## Key Concepts and Terminology

### Buckets
- **Definition**: A container for objects stored in S3
- **Naming**: Must be globally unique across all AWS accounts
- **Location**: Created in a specific AWS region
- **Limit**: 100 buckets per account by default

### Objects
- **Definition**: Files stored in S3 buckets
- **Size**: 0 bytes to 5TB per object
- **Key**: Unique identifier within a bucket
- **Metadata**: Key-value pairs describing the object

### Regions and Availability Zones
- Data is stored in specific AWS regions
- Automatically replicated across multiple Availability Zones
- Choose regions based on latency, compliance, and cost

## S3 Storage Classes

### Standard Storage Classes
1. **S3 Standard**
   - Default storage class
   - High durability and availability
   - Use case: Frequently accessed data

2. **S3 Standard-IA (Infrequent Access)**
   - Lower cost than Standard
   - Retrieval fees apply
   - Use case: Data accessed less frequently but requires rapid access

3. **S3 One Zone-IA**
   - Lower cost than Standard-IA
   - Stored in single AZ
   - Use case: Recreatable data with infrequent access

### Archive Storage Classes
4. **S3 Glacier Instant Retrieval**
   - Archive data with millisecond retrieval
   - Use case: Archive data accessed once per quarter

5. **S3 Glacier Flexible Retrieval**
   - Low-cost archive storage
   - Retrieval: minutes to hours
   - Use case: Backup and archive

6. **S3 Glacier Deep Archive**
   - Lowest cost storage
   - Retrieval: 12+ hours
   - Use case: Long-term archive and digital preservation

7. **S3 Intelligent-Tiering**
   - Automatically moves data between tiers
   - No retrieval fees
   - Use case: Unknown or changing access patterns

## S3 Security Features

### Access Control
- **Bucket Policies**: JSON-based access policies
- **IAM Policies**: User and role-based permissions
- **Access Control Lists (ACLs)**: Object-level permissions
- **Pre-signed URLs**: Temporary access to objects

### Encryption
- **Server-Side Encryption (SSE)**
  - SSE-S3: S3-managed keys
  - SSE-KMS: AWS KMS-managed keys
  - SSE-C: Customer-provided keys
- **Client-Side Encryption**: Encrypt before upload

### Other Security Features
- **MFA Delete**: Require MFA for object deletion
- **Logging**: CloudTrail and S3 access logs
- **VPC Endpoints**: Private connectivity from VPC

## S3 Versioning and Lifecycle

### Versioning
- Keep multiple versions of objects
- Protect against accidental deletion
- Each version has unique version ID

### Lifecycle Management
- Automatically transition objects between storage classes
- Delete objects after specified time
- Rules based on prefixes and tags

## Creating S3 Buckets with Infrastructure as Code

### Method 1: AWS CloudFormation Template (CFT)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Complete S3 Bucket Configuration'

Parameters:
  BucketName:
    Type: String
    Description: Name of the S3 bucket
    Default: my-devops-bucket-12345
  
  Environment:
    Type: String
    Description: Environment name
    Default: dev
    AllowedValues:
      - dev
      - staging
      - prod

Resources:
  # Main S3 Bucket
  S3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub '${BucketName}-${Environment}'
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      VersioningConfiguration:
        Status: Enabled
      LifecycleConfiguration:
        Rules:
          - Id: DeleteIncompleteMultipartUploads
            Status: Enabled
            AbortIncompleteMultipartUpload:
              DaysAfterInitiation: 7
          - Id: TransitionToIA
            Status: Enabled
            Transition:
              StorageClass: STANDARD_IA
              TransitionInDays: 30
          - Id: TransitionToGlacier
            Status: Enabled
            Transition:
              StorageClass: GLACIER
              TransitionInDays: 90
          - Id: DeleteOldVersions
            Status: Enabled
            NoncurrentVersionExpiration:
              NoncurrentDays: 365
      NotificationConfiguration:
        CloudWatchConfigurations:
          - Event: s3:ObjectCreated:*
            CloudWatchConfiguration:
              LogGroupName: !Ref S3LogGroup
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      Tags:
        - Key: Environment
          Value: !Ref Environment
        - Key: Purpose
          Value: DevOps Learning
        - Key: Owner
          Value: DevOps Team

  # Bucket Policy for specific access
  S3BucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref S3Bucket
      PolicyDocument:
        Statement:
          - Sid: DenyInsecureConnections
            Effect: Deny
            Principal: '*'
            Action: 's3:*'
            Resource:
              - !Sub '${S3Bucket}/*'
              - !Ref S3Bucket
            Condition:
              Bool:
                'aws:SecureTransport': 'false'
          - Sid: AllowCloudFrontAccess
            Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity ${CloudFrontOAI}'
            Action: 's3:GetObject'
            Resource: !Sub '${S3Bucket}/*'

  # CloudFront Origin Access Identity
  CloudFrontOAI:
    Type: AWS::CloudFront::CloudFrontOriginAccessIdentity
    Properties:
      CloudFrontOriginAccessIdentityConfig:
        Comment: !Sub 'OAI for ${BucketName}'

  # CloudWatch Log Group for S3 notifications
  S3LogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub '/aws/s3/${BucketName}-${Environment}'
      RetentionInDays: 14

  # IAM Role for S3 operations
  S3AccessRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub 'S3-Access-Role-${Environment}'
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: S3AccessPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - 's3:GetObject'
                  - 's3:PutObject'
                  - 's3:DeleteObject'
                  - 's3:ListBucket'
                Resource:
                  - !Sub '${S3Bucket}/*'
                  - !Ref S3Bucket

Outputs:
  BucketName:
    Description: Name of the created S3 bucket
    Value: !Ref S3Bucket
    Export:
      Name: !Sub '${AWS::StackName}-BucketName'
  
  BucketArn:
    Description: ARN of the created S3 bucket
    Value: !GetAtt S3Bucket.Arn
    Export:
      Name: !Sub '${AWS::StackName}-BucketArn'
  
  BucketDomainName:
    Description: Domain name of the S3 bucket
    Value: !GetAtt S3Bucket.DomainName
    Export:
      Name: !Sub '${AWS::StackName}-BucketDomainName'
```

### Method 2: Terraform Configuration

```hcl
# variables.tf
variable "bucket_name" {
  description = "Name of the S3 bucket"
  type        = string
  default     = "my-devops-bucket"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

# main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region
}

# Random ID for bucket naming
resource "random_id" "bucket_suffix" {
  byte_length = 4
}

# Main S3 Bucket
resource "aws_s3_bucket" "main_bucket" {
  bucket = "${var.bucket_name}-${var.environment}-${random_id.bucket_suffix.hex}"

  tags = {
    Name        = "${var.bucket_name}-${var.environment}"
    Environment = var.environment
    Purpose     = "DevOps Learning"
    Owner       = "DevOps Team"
  }
}

# Bucket Versioning
resource "aws_s3_bucket_versioning" "main_bucket_versioning" {
  bucket = aws_s3_bucket.main_bucket.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Bucket Encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "main_bucket_encryption" {
  bucket = aws_s3_bucket.main_bucket.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
    bucket_key_enabled = true
  }
}

# Public Access Block
resource "aws_s3_bucket_public_access_block" "main_bucket_pab" {
  bucket = aws_s3_bucket.main_bucket.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Lifecycle Configuration
resource "aws_s3_bucket_lifecycle_configuration" "main_bucket_lifecycle" {
  bucket = aws_s3_bucket.main_bucket.id

  rule {
    id     = "delete_incomplete_multipart_uploads"
    status = "Enabled"

    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }

  rule {
    id     = "transition_to_ia"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }
  }

  rule {
    id     = "transition_to_glacier"
    status = "Enabled"

    transition {
      days          = 90
      storage_class = "GLACIER"
    }
  }

  rule {
    id     = "delete_old_versions"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days = 365
    }
  }
}

# Bucket Policy
resource "aws_s3_bucket_policy" "main_bucket_policy" {
  bucket = aws_s3_bucket.main_bucket.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyInsecureConnections"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = [
          aws_s3_bucket.main_bucket.arn,
          "${aws_s3_bucket.main_bucket.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      }
    ]
  })
}

# CloudWatch Log Group
resource "aws_cloudwatch_log_group" "s3_log_group" {
  name              = "/aws/s3/${aws_s3_bucket.main_bucket.bucket}"
  retention_in_days = 14

  tags = {
    Environment = var.environment
    Purpose     = "S3 Bucket Logging"
  }
}

# IAM Role for S3 Access
resource "aws_iam_role" "s3_access_role" {
  name = "S3-Access-Role-${var.environment}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })

  tags = {
    Environment = var.environment
    Purpose     = "S3 Access"
  }
}

# IAM Policy for S3 Access
resource "aws_iam_role_policy" "s3_access_policy" {
  name = "S3AccessPolicy"
  role = aws_iam_role.s3_access_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.main_bucket.arn,
          "${aws_s3_bucket.main_bucket.arn}/*"
        ]
      }
    ]
  })
}

# outputs.tf
output "bucket_name" {
  description = "Name of the created S3 bucket"
  value       = aws_s3_bucket.main_bucket.bucket
}

output "bucket_arn" {
  description = "ARN of the created S3 bucket"
  value       = aws_s3_bucket.main_bucket.arn
}

output "bucket_domain_name" {
  description = "Domain name of the S3 bucket"
  value       = aws_s3_bucket.main_bucket.bucket_domain_name
}

output "iam_role_arn" {
  description = "ARN of the IAM role for S3 access"
  value       = aws_iam_role.s3_access_role.arn
}
```

## Real-World Examples

### Example 1: Static Website Hosting

```yaml
# CloudFormation for Static Website
Resources:
  WebsiteBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub 'my-website-${AWS::AccountId}'
      WebsiteConfiguration:
        IndexDocument: index.html
        ErrorDocument: error.html
      PublicAccessBlockConfiguration:
        BlockPublicAcls: false
        BlockPublicPolicy: false
        IgnorePublicAcls: false
        RestrictPublicBuckets: false

  WebsiteBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref WebsiteBucket
      PolicyDocument:
        Statement:
          - Sid: PublicReadGetObject
            Effect: Allow
            Principal: '*'
            Action: 's3:GetObject'
            Resource: !Sub '${WebsiteBucket}/*'
```

### Example 2: Data Lake Architecture

```hcl
# Terraform for Data Lake
resource "aws_s3_bucket" "data_lake" {
  bucket = "company-data-lake-${random_id.suffix.hex}"
}

resource "aws_s3_bucket_notification" "data_lake_notification" {
  bucket = aws_s3_bucket.data_lake.id

  lambda_function {
    lambda_function_arn = aws_lambda_function.data_processor.arn
    events              = ["s3:ObjectCreated:*"]
    filter_prefix       = "raw-data/"
    filter_suffix       = ".json"
  }
}

# Lambda function for data processing
resource "aws_lambda_function" "data_processor" {
  filename         = "data_processor.zip"
  function_name    = "s3-data-processor"
  role            = aws_iam_role.lambda_role.arn
  handler         = "index.handler"
  runtime         = "python3.9"
}
```

### Example 3: Backup and Archive Solution

```yaml
# CloudFormation for Backup Solution
Resources:
  BackupBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub 'backup-bucket-${AWS::AccountId}'
      LifecycleConfiguration:
        Rules:
          - Id: BackupLifecycle
            Status: Enabled
            Transitions:
              - StorageClass: STANDARD_IA
                TransitionInDays: 30
              - StorageClass: GLACIER
                TransitionInDays: 90
              - StorageClass: DEEP_ARCHIVE
                TransitionInDays: 365
            ExpirationInDays: 2555  # 7 years
      ReplicationConfiguration:
        Role: !GetAtt ReplicationRole.Arn
        Rules:
          - Id: ReplicateToSecondaryRegion
            Status: Enabled
            Prefix: critical/
            Destination:
              Bucket: !Sub 'arn:aws:s3:::backup-bucket-replica-${AWS::AccountId}'
              StorageClass: STANDARD_IA
```

### Example 4: Content Distribution with CloudFront

```hcl
# Terraform for CDN setup
resource "aws_s3_bucket" "content_bucket" {
  bucket = "my-content-bucket-${random_id.suffix.hex}"
}

resource "aws_cloudfront_distribution" "content_distribution" {
  origin {
    domain_name = aws_s3_bucket.content_bucket.bucket_regional_domain_name
    origin_id   = "S3-${aws_s3_bucket.content_bucket.bucket}"

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.oai.cloudfront_access_identity_path
    }
  }

  enabled             = true
  default_root_object = "index.html"

  default_cache_behavior {
    allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id      = "S3-${aws_s3_bucket.content_bucket.bucket}"
    compress              = true
    viewer_protocol_policy = "redirect-to-https"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}
```

## Best Practices

### Naming Conventions
- Use lowercase letters, numbers, and hyphens
- Include environment and purpose in name
- Avoid using periods in bucket names
- Keep names descriptive but concise

### Security Best Practices
1. **Enable encryption by default**
2. **Block public access unless required**
3. **Use IAM policies for access control**
4. **Enable CloudTrail logging**
5. **Use VPC endpoints for private access**
6. **Implement MFA delete for critical data**

### Performance Optimization
1. **Use appropriate storage classes**
2. **Implement lifecycle policies**
3. **Use multipart upload for large files**
4. **Optimize request patterns**
5. **Use CloudFront for global distribution**

### Cost Optimization
1. **Use Intelligent-Tiering for unknown access patterns**
2. **Implement lifecycle policies**
3. **Delete incomplete multipart uploads**
4. **Monitor usage with Cost Explorer**
5. **Use S3 Storage Lens for insights**

## Monitoring and Logging

### CloudWatch Metrics
- **BucketSizeBytes**: Total size of bucket
- **NumberOfObjects**: Count of objects
- **AllRequests**: Total number of requests
- **GetRequests**: Number of GET requests
- **PutRequests**: Number of PUT requests

### S3 Access Logging
```yaml
# Enable access logging
AccessLoggingBucket:
  Type: AWS::S3::Bucket
  Properties:
    BucketName: !Sub '${MainBucket}-access-logs'
    LifecycleConfiguration:
      Rules:
        - Id: DeleteAccessLogs
          Status: Enabled
          ExpirationInDays: 90

MainBucket:
  Type: AWS::S3::Bucket
  Properties:
    LoggingConfiguration:
      DestinationBucketName: !Ref AccessLoggingBucket
      LogFilePrefix: access-logs/
```

### CloudTrail Integration
```hcl
resource "aws_cloudtrail" "s3_trail" {
  name           = "s3-api-trail"
  s3_bucket_name = aws_s3_bucket.trail_bucket.bucket

  event_selector {
    read_write_type                 = "All"
    include_management_events       = false
    data_resource {
      type   = "AWS::S3::Object"
      values = ["${aws_s3_bucket.main_bucket.arn}/*"]
    }
  }
}
```

## Cost Optimization

### Storage Class Analysis
```yaml
# Enable storage class analysis
StorageClassAnalysis:
  Type: AWS::S3::Bucket
  Properties:
    AnalyticsConfigurations:
      - Id: EntireBucket
        StorageClassAnalysis:
          DataExport:
            Destination:
              BucketArn: !GetAtt AnalyticsBucket.Arn
              Format: CSV
              Prefix: storage-analysis/
            OutputSchemaVersion: V_1
```

### Intelligent Tiering
```hcl
resource "aws_s3_bucket_intelligent_tiering_configuration" "intelligent_tiering" {
  bucket = aws_s3_bucket.main_bucket.bucket
  name   = "EntireBucket"

  filter {
    prefix = ""
  }

  tiering {
    access_tier = "DEEP_ARCHIVE_ACCESS"
    days        = 180
  }

  tiering {
    access_tier = "ARCHIVE_ACCESS"
    days        = 125
  }
}
```

## Deployment Commands

### CloudFormation Deployment
```bash
# Deploy the stack
aws cloudformation create-stack \
  --stack-name s3-devops-stack \
  --template-body file://s3-template.yaml \
  --parameters ParameterKey=BucketName,ParameterValue=my-devops-bucket \
               ParameterKey=Environment,ParameterValue=dev \
  --capabilities CAPABILITY_IAM

# Update the stack
aws cloudformation update-stack \
  --stack-name s3-devops-stack \
  --template-body file://s3-template.yaml \
  --parameters ParameterKey=BucketName,ParameterValue=my-devops-bucket \
               ParameterKey=Environment,ParameterValue=dev \
  --capabilities CAPABILITY_IAM

# Delete the stack
aws cloudformation delete-stack --stack-name s3-devops-stack
```

### Terraform Deployment
```bash
# Initialize Terraform
terraform init

# Plan the deployment
terraform plan -var="bucket_name=my-devops-bucket" -var="environment=dev"

# Apply the configuration
terraform apply -var="bucket_name=my-devops-bucket" -var="environment=dev"

# Destroy the infrastructure
terraform destroy -var="bucket_name=my-devops-bucket" -var="environment=dev"
```

## Conclusion

AWS S3 is a fundamental service in the AWS ecosystem that provides reliable, scalable, and secure object storage. Understanding S3 concepts, security features, and best practices is crucial for any DevOps engineer. The Infrastructure as Code examples provided here demonstrate how to create production-ready S3 configurations that follow AWS best practices.

Key takeaways from today's learning:
- S3 provides multiple storage classes for different use cases
- Security should be implemented at multiple layers
- Lifecycle policies help optimize costs automatically
- Infrastructure as Code ensures consistent and repeatable deployments
- Monitoring and logging are essential for operational excellence

Continue practicing with these examples and explore additional S3 features like Cross-Region Replication, Transfer Acceleration, and integration with other AWS services.

---
*Happy Learning! 🚀*