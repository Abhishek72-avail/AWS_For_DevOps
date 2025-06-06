# AWS Route 53 - Complete Guide

## Table of Contents
- [What is AWS Route 53?](#what-is-aws-route-53)
- [Key Features](#key-features)
- [DNS Record Types](#dns-record-types)
- [Getting Started](#getting-started)
- [Step-by-Step Setup](#step-by-step-setup)
- [Routing Policies](#routing-policies)
- [Health Checks and Monitoring](#health-checks-and-monitoring)
- [Advanced Features](#advanced-features)
- [Real-World Use Cases](#real-world-use-cases)
- [Pricing](#pricing)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

## What is AWS Route 53?

AWS Route 53 is a highly scalable and reliable **Domain Name System (DNS)** web service designed to route end users to internet applications. It translates human-readable domain names (like `example.com`) into IP addresses (like `192.0.2.1`) that computers use to connect to each other.

### Why "Route 53"?
The name comes from TCP/UDP port 53, which is the standard port used for DNS queries.

## Key Features

### 🌐 **Domain Registration**
- Register new domain names directly through Route 53
- Transfer existing domains from other registrars
- Automatic renewal options

### 🗺️ **DNS Management**
- Authoritative DNS service for your domains
- Support for all common DNS record types
- Global network of DNS servers

### 🎯 **Traffic Routing**
- Multiple routing policies for traffic distribution
- Geographic and latency-based routing
- Weighted and failover routing

### 🏥 **Health Monitoring**
- Health checks for endpoints
- Automatic failover capabilities
- CloudWatch integration

### 🔒 **Security & Compliance**
- DNSSEC support
- Private DNS for VPCs
- AWS IAM integration

## DNS Record Types

| Record Type | Purpose | Example |
|-------------|---------|---------|
| **A** | Maps domain to IPv4 address | `example.com → 192.0.2.1` |
| **AAAA** | Maps domain to IPv6 address | `example.com → 2001:db8::1` |
| **CNAME** | Creates alias for another domain | `www.example.com → example.com` |
| **MX** | Mail exchange records | `example.com → mail.example.com` |
| **TXT** | Text records (verification, SPF) | `example.com → "v=spf1 include:_spf.google.com ~all"` |
| **NS** | Name server records | `example.com → ns1.route53.amazonaws.com` |
| **SOA** | Start of Authority | Defines authoritative information |
| **PTR** | Reverse DNS lookup | `1.2.0.192.in-addr.arpa → example.com` |
| **SRV** | Service location | `_sip._tcp.example.com` |

## Getting Started

### Prerequisites
- AWS Account with appropriate permissions
- Domain name (can register through Route 53 or transfer existing)
- Basic understanding of DNS concepts

### Required AWS Permissions
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "route53:*",
                "route53domains:*",
                "cloudwatch:*",
                "health:*"
            ],
            "Resource": "*"
        }
    ]
}
```

## Step-by-Step Setup

### 1. Register or Transfer a Domain

#### Option A: Register New Domain
```bash
# Using AWS CLI
aws route53domains register-domain \
    --domain-name example.com \
    --duration-in-years 1 \
    --admin-contact file://admin-contact.json \
    --registrant-contact file://registrant-contact.json \
    --tech-contact file://tech-contact.json
```

#### Option B: Transfer Existing Domain
1. **Unlock domain** at current registrar
2. **Get authorization code** from current registrar
3. **Initiate transfer** in Route 53 console
4. **Confirm transfer** via email

### 2. Create a Hosted Zone

```bash
# Create hosted zone
aws route53 create-hosted-zone \
    --name example.com \
    --caller-reference $(date +%s)
```

**Console Steps:**
1. Navigate to Route 53 → Hosted zones
2. Click "Create hosted zone"
3. Enter domain name
4. Select "Public hosted zone"
5. Click "Create hosted zone"

### 3. Configure DNS Records

#### Basic A Record
```bash
# Create A record
aws route53 change-resource-record-sets \
    --hosted-zone-id Z123456789 \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "example.com",
                "Type": "A",
                "TTL": 300,
                "ResourceRecords": [{"Value": "192.0.2.1"}]
            }
        }]
    }'
```

#### WWW CNAME Record
```bash
# Create CNAME record
aws route53 change-resource-record-sets \
    --hosted-zone-id Z123456789 \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "www.example.com",
                "Type": "CNAME",
                "TTL": 300,
                "ResourceRecords": [{"Value": "example.com"}]
            }
        }]
    }'
```

### 4. Update Name Servers

If you registered domain elsewhere, update name servers to Route 53:
```
ns-123.awsdns-12.com
ns-456.awsdns-45.net
ns-789.awsdns-78.org
ns-012.awsdns-01.co.uk
```

## Routing Policies

### 1. **Simple Routing**
- Default routing policy
- One resource per record
- No health checks

```json
{
    "Name": "example.com",
    "Type": "A",
    "TTL": 300,
    "ResourceRecords": [{"Value": "192.0.2.1"}]
}
```

### 2. **Weighted Routing**
- Distribute traffic based on weights
- Useful for A/B testing

```bash
# 70% traffic to server 1
aws route53 change-resource-record-sets \
    --hosted-zone-id Z123456789 \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "example.com",
                "Type": "A",
                "SetIdentifier": "server-1",
                "Weight": 70,
                "TTL": 300,
                "ResourceRecords": [{"Value": "192.0.2.1"}]
            }
        }]
    }'

# 30% traffic to server 2
aws route53 change-resource-record-sets \
    --hosted-zone-id Z123456789 \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "example.com",
                "Type": "A",
                "SetIdentifier": "server-2",
                "Weight": 30,
                "TTL": 300,
                "ResourceRecords": [{"Value": "192.0.2.2"}]
            }
        }]
    }'
```

### 3. **Latency-Based Routing**
- Route to region with lowest latency

```json
{
    "Name": "example.com",
    "Type": "A",
    "SetIdentifier": "us-east-1",
    "Region": "us-east-1",
    "TTL": 300,
    "ResourceRecords": [{"Value": "192.0.2.1"}]
}
```

### 4. **Failover Routing**
- Primary/secondary setup for disaster recovery

```json
{
    "Name": "example.com",
    "Type": "A",
    "SetIdentifier": "primary",
    "Failover": "PRIMARY",
    "TTL": 300,
    "ResourceRecords": [{"Value": "192.0.2.1"}],
    "HealthCheckId": "health-check-id"
}
```

### 5. **Geolocation Routing**
- Route based on user's geographic location

```json
{
    "Name": "example.com",
    "Type": "A",
    "SetIdentifier": "europe",
    "GeoLocation": {
        "ContinentCode": "EU"
    },
    "TTL": 300,
    "ResourceRecords": [{"Value": "203.0.113.1"}]
}
```

### 6. **Multivalue Answer**
- Return multiple IP addresses with health checks

```json
{
    "Name": "example.com",
    "Type": "A",
    "SetIdentifier": "server-1",
    "TTL": 300,
    "ResourceRecords": [{"Value": "192.0.2.1"}],
    "HealthCheckId": "health-check-1"
}
```

## Health Checks and Monitoring

### Creating Health Checks

```bash
# HTTP health check
aws route53 create-health-check \
    --caller-reference $(date +%s) \
    --health-check-config '{
        "Type": "HTTP",
        "ResourcePath": "/health",
        "FullyQualifiedDomainName": "example.com",
        "Port": 80,
        "RequestInterval": 30,
        "FailureThreshold": 3
    }'
```

### Health Check Types

1. **HTTP/HTTPS Checks**
   - Monitor web endpoints
   - Custom ports and paths
   - String matching

2. **TCP Checks**
   - Monitor any TCP service
   - Database connections
   - Custom applications

3. **Calculated Checks**
   - Combine multiple health checks
   - Boolean logic (AND, OR)

4. **CloudWatch Alarm Checks**
   - Monitor AWS resources
   - Custom metrics

### Health Check Configuration

```json
{
    "Type": "HTTPS",
    "ResourcePath": "/api/health",
    "FullyQualifiedDomainName": "api.example.com",
    "Port": 443,
    "RequestInterval": 30,
    "FailureThreshold": 3,
    "MeasureLatency": true,
    "EnableSNI": true,
    "SearchString": "OK"
}
```

## Advanced Features

### 1. **Private DNS**
- DNS resolution within VPC
- Split-horizon DNS

```bash
# Create private hosted zone
aws route53 create-hosted-zone \
    --name internal.example.com \
    --caller-reference $(date +%s) \
    --vpc VPCRegion=us-east-1,VPCId=vpc-123456
```

### 2. **DNSSEC**
- DNS Security Extensions
- Protect against cache poisoning

```bash
# Enable DNSSEC
aws route53 enable-hosted-zone-dnssec \
    --hosted-zone-id Z123456789
```

### 3. **Traffic Flow**
- Visual traffic routing policies
- Complex routing scenarios

### 4. **Resolver**
- Hybrid DNS resolution
- On-premises integration

```bash
# Create resolver endpoint
aws route53resolver create-resolver-endpoint \
    --creator-request-id $(date +%s) \
    --security-group-ids sg-123456 \
    --direction INBOUND \
    --ip-addresses SubnetId=subnet-123456,Ip=10.0.1.5
```

## Real-World Use Cases

### 1. **E-commerce Platform**
```
┌─────────────────┐    ┌──────────────────┐
│   Users         │───▶│  Route 53        │
│                 │    │  Geographic      │
└─────────────────┘    │  Routing         │
                       └──────────────────┘
                              │
               ┌──────────────┼──────────────┐
               ▼              ▼              ▼
    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
    │   US East   │  │   Europe    │  │   Asia      │
    │   Servers   │  │   Servers   │  │   Servers   │
    └─────────────┘  └─────────────┘  └─────────────┘
```

**Implementation:**
- **Geolocation routing** for regional servers
- **Health checks** for automatic failover
- **Weighted routing** for gradual deployments

### 2. **Disaster Recovery Setup**
```
Primary Site (Active)     Secondary Site (Standby)
┌─────────────────┐      ┌─────────────────┐
│   us-east-1     │      │   us-west-2     │
│   Web Servers   │      │   Web Servers   │
│   Database      │      │   Database      │
└─────────────────┘      └─────────────────┘
         │                        │
         └────────┬─────────────── ┘
                  ▼
           ┌─────────────┐
           │  Route 53   │
           │  Failover   │
           │  Routing    │
           └─────────────┘
```

**Configuration:**
```json
{
    "Primary": {
        "Failover": "PRIMARY",
        "HealthCheckId": "primary-health-check"
    },
    "Secondary": {
        "Failover": "SECONDARY"
    }
}
```

### 3. **Microservices Architecture**
```
api.example.com
├── /users     → users-service.internal
├── /products  → products-service.internal
├── /orders    → orders-service.internal
└── /payments  → payments-service.internal
```

**DNS Records:**
```
api.example.com          → Load Balancer
users.internal.com       → User Service Instances
products.internal.com    → Product Service Instances
orders.internal.com      → Order Service Instances
```

### 4. **CDN Integration**
```bash
# CloudFront distribution
www.example.com → CNAME → d123.cloudfront.net

# Regional optimization
cdn-us.example.com    → US CloudFront Edge
cdn-eu.example.com    → EU CloudFront Edge
cdn-asia.example.com  → Asia CloudFront Edge
```

### 5. **Blue-Green Deployment**
```
Production Environment:
blue.example.com  (Weight: 100)
green.example.com (Weight: 0)

During Deployment:
blue.example.com  (Weight: 50)
green.example.com (Weight: 50)

After Validation:
blue.example.com  (Weight: 0)
green.example.com (Weight: 100)
```

### 6. **Multi-Region Application**
```yaml
Routing Strategy:
- North America → us-east-1
- Europe → eu-west-1
- Asia Pacific → ap-southeast-1
- Default → us-east-1

Health Checks:
- Monitor each region
- Automatic failover to healthy regions
```

## Pricing

### Domain Registration
- **.com domains:** $12/year
- **.org domains:** $12.20/year
- **.net domains:** $11.50/year
- **Country-specific:** Varies

### DNS Queries
- **First 1 billion queries/month:** $0.40 per million
- **Over 1 billion queries/month:** $0.20 per million

### Health Checks
- **Basic health checks:** $0.50/month each
- **Health checks with string matching:** $1.00/month each
- **Calculated health checks:** $1.00/month each

### Hosted Zones
- **First 25 hosted zones:** $0.50/month each
- **Additional hosted zones:** $0.10/month each

## Best Practices

### 1. **TTL Management**
```bash
# Short TTL for frequent changes
TTL=60    # 1 minute

# Long TTL for stable records
TTL=86400 # 24 hours
```

### 2. **Health Check Strategy**
- Monitor application endpoints, not just servers
- Use multiple health check locations
- Set appropriate failure thresholds
- Test health check endpoints regularly

### 3. **Security**
```json
{
    "DNSSEC": "Enable for critical domains",
    "PrivateDNS": "Use for internal services",
    "IAM": "Restrict Route 53 permissions",
    "Logging": "Enable query logging"
}
```

### 4. **Monitoring and Alerting**
```bash
# CloudWatch metrics to monitor
- QueryCount
- HealthCheckStatus
- HealthCheckPercentHealthy
- ConnectionTime
```

### 5. **Backup and Recovery**
```bash
# Export zone file
aws route53 list-resource-record-sets \
    --hosted-zone-id Z123456789 \
    --output text > zone-backup.txt

# Regular backups of DNS configurations
# Version control for DNS changes
# Test disaster recovery procedures
```

## Troubleshooting

### Common Issues

#### 1. **DNS Resolution Problems**
```bash
# Test DNS resolution
nslookup example.com
dig example.com
host example.com

# Check propagation
dig @8.8.8.8 example.com
dig @1.1.1.1 example.com
```

#### 2. **Health Check Failures**
```bash
# Debug health check
aws route53 get-health-check --health-check-id HXXXXXXXXX
aws route53 get-health-check-status --health-check-id HXXXXXXXXX

# Common causes:
- Firewall blocking health check IPs
- Incorrect endpoint configuration
- SSL certificate issues
- Application returning wrong status codes
```

#### 3. **Routing Policy Issues**
```bash
# Verify record sets
aws route53 list-resource-record-sets \
    --hosted-zone-id Z123456789

# Check weights and identifiers
# Validate geographic locations
# Confirm health check associations
```

### Health Check IP Ranges
```
# Allow these IP ranges for health checks
54.228.16.0/26
54.232.40.64/26
107.23.255.0/26
# ... (full list in AWS documentation)
```

### Debugging Commands
```bash
# Route 53 query logging
aws logs create-log-group --log-group-name /aws/route53/queries

# DNS trace
dig +trace example.com

# Check nameservers
dig NS example.com

# Verify SOA record
dig SOA example.com
```

## Conclusion

AWS Route 53 is a powerful DNS service that goes beyond basic domain resolution. With features like intelligent routing, health monitoring, and seamless AWS integration, it's essential for building resilient, globally distributed applications.

Key takeaways:
- Start with simple DNS records and gradually add advanced features
- Always implement health checks for critical applications
- Use appropriate routing policies for your use case
- Monitor DNS performance and set up alerting
- Regular backup and testing of DNS configurations

## Additional Resources

- [AWS Route 53 Documentation](https://docs.aws.amazon.com/route53/)
- [Route 53 API Reference](https://docs.aws.amazon.com/Route53/latest/APIReference/)
- [DNS Best Practices](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/best-practices-dns.html)
- [Route 53 Pricing](https://aws.amazon.com/route53/pricing/)

---

**Happy DNS management with Route 53! 🚀**