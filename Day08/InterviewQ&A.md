Scenario-Based Interview Questions on EC2, IAM, and VPC
1. Designing a VPC Architecture for a 2-Tier Application
Q: You have been assigned to design a VPC architecture for a 2-tier application. The application needs to be highly available and scalable. How would you design the VPC architecture?

A:
I would create 2 subnets: public and private. The public subnet would contain the load balancers and be accessible from the internet. The private subnet would host the application servers. I would distribute the subnets across multiple Availability Zones for high availability. Additionally, I would configure auto scaling groups for the application servers.

2. Restricting Outbound Internet Access for Specific Subnets
Q: Your organization has a VPC with multiple subnets. You want to restrict outbound internet access for resources in one subnet but allow it for another. How would you achieve this?

A:
To restrict outbound internet access for one subnet, modify its route table to remove the default route (0.0.0.0/0) pointing to the internet gateway. For the subnet requiring internet access, keep the default route pointing to the internet gateway.

3. Allowing Private Subnet Instances to Access the Internet
Q: You have a VPC with a public subnet and a private subnet. Instances in the private subnet need internet access for software updates. How would you allow internet access for these instances?

A:
Use a NAT Gateway or NAT instance in the public subnet. Configure the private subnet’s route table to direct outbound traffic to the NAT Gateway/instance, enabling internet access.

4. Enabling Communication Between EC2 Instances in a VPC
Q: You have launched EC2 instances in your VPC and want them to communicate with each other using private IPs. What steps would you take?

A:
By default, instances within the same VPC can communicate using private IPs. Ensure instances are in the same VPC and subnets (or connected via VPC peering if in different VPCs). Check security groups to allow necessary inbound/outbound traffic.

5. Implementing Strict Network Access Control
Q: You want to implement strict network access control for your VPC resources. How would you achieve this?

A:
Use Network ACLs (NACLs) at the subnet level to define inbound/outbound rules based on source/destination IPs, ports, and protocols. NACLs are stateless and can enforce fine-grained access control for traffic entering and leaving subnets.

6. Creating an Isolated Environment for Sensitive Workloads
Q: Your organization requires an isolated environment within the VPC for running sensitive workloads. How would you set up this isolated environment?

A:
Create a subnet with no internet gateway attached (an “isolated subnet”). Place sensitive workloads there, ensuring no direct internet connectivity. If outbound access is needed, use a NAT Gateway or NAT instance in a different subnet and adjust route tables accordingly.

7. Accessing AWS Services Securely Within a VPC
Q: Your application needs to access AWS services like S3 securely within your VPC. How would you achieve this?

A:
Use VPC endpoints for services like S3 and DynamoDB. VPC endpoints allow private connectivity between VPC resources and AWS services without requiring an internet gateway or NAT Gateway.

8. NACLs vs. Security Groups
Q: What is the difference between NACLs and Security Groups? Explain with a use case.

A:

NACLs: Operate at the subnet level, are stateless, and filter inbound/outbound traffic based on source/destination IPs, ports, and protocols.

Security Groups: Operate at the instance level, are stateful, and control inbound/outbound traffic based on application needs.

Use Case:
Use NACLs for subnet-level filtering (additional defense), and security groups for controlling traffic to/from individual instances for application-level security. Together, they provide defense-in-depth.

9. IAM Users, Groups, Roles, and Policies
Q: What is the difference between IAM users, groups, roles, and policies?

A:

IAM User: Represents an individual/application with long-term credentials. Can be assigned directly to policies or added to groups.

IAM Group: Collection of users to manage permissions collectively.

IAM Role: Not linked to a single user; assumed by users/services for temporary credentials, useful for cross-account access.

IAM Policy: JSON document defining permissions/actions on resources. Attached to users, groups, or roles.

10. Secure Administrative Access to Private Subnet Instances
Q: You have a private subnet with instances that should not have direct internet access, but you still need to administer them. How would you set up a bastion host?

A:

Deploy a bastion host (jump box) EC2 instance in a public subnet with a public or Elastic IP.

Configure the bastion host’s security group to allow inbound SSH/RDP from trusted IPs.

Configure private subnet instances’ security groups to allow SSH/RDP from the bastion host.

SSH/RDP into the bastion host, then connect to private instances using their private IPs.