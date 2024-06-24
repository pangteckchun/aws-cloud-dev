# Architecting on AWS - Jun 2024 #
Need a AWS builder ID: teckchun.pang@gmail.com  
Need a vital source ID for training materials  
Lab 2 to 6 are important exercises to pass the capstone project (lab 7)

---
## Module #1 - Architecting Fundamentals ##

### Summary ###
- Categories in focus for this course: Serverless, Networking, DB, Security, governance, storage, costs management, compute, containers.
- 1 AZ - multi DCs (different from what i know prev).
- 1 region - at least 3 AZs (typically 50 to 100km apart), interconnected with high speed fiber.
- Regions are isolated from each other.
- Select region based on: costs, types of services availability, latency, data regulations (data residency etc).
- `AWS Local Zones`: available in Bangkok and Manila only | a separate mini DC outside of the AZs in that region in a busy area, e.g. in a CBD | for dedicated and super low latency requirements (real time gaming, live video streaming, AR/VR) | AWS infra at the edge with small number of services.
- `Edge locations`: combo of CDNs (AWS Cloudfront) and routing (Route 53) | storage and cache services for near-shore data access only without processing services | only for hot data state and not all data are fetched and cached | still need to fetch from source to edge location for the first user.
- `AWS Well Architected Framework` 6 pillars: Security, Performance efficiency, Cost optimization, Operational excellence, Reliability, Sustainability.

### Lab #1 - Using AWS CLI for s3 ###
- Listing all buckets in s3 - `aws s3 ls`
- Creating a new bucket - `aws s3 mb s3://labclibucket-NUMBER`
- Copying a local file to s3 - `aws s3 cp /home/ssm-user/HappyFace.jpg s3://labclibucket-NUMBER`
- List what is in a s3 bucket - `aws s3 ls s3://labclibucket-NUMBER`
---

## Module #2 - Account Security ##
**Root** user is for IAM provisioning and emergency cases; do not use for daily interactions.  
Root is your default HPA account; anything else created in `IAM` are *sub-accounts*.

### IAM ###
- Always create a separate sub-account / ID for every individual.
- Access via CLI or SDK always need `access key` and `secret access key` pairs.
- Use **user groups** to manage pool of users with similar authorization.
- User **IAM role** for temp permissions (time validity) for individual users, services, federated users.
- A **user group** cannot assume an **IAM role**.
- `IAM policies` are Json documents with **version**, **effect**, **action** (Allow or Deny only), **resource** accessible, **conditions** the policy will apply.
- Effect - Allow or Deny only. Deny takes precedence over Allow.
- `IAM policies` can be identity-based or resource-based: different Json formats. Both **identity-based** and **resource-based** IAM policies will be evaluated at the same to reconcile access conflicts.  
-- Resource-based policy will always have "principal" as an attribute. `IAM Permission Boundaries`: sets the range or limits of permissions we can assign out in our policies. This sets the filter or boundaries for max permission. It does not *grant* permission; use `IAM policies` to do the granting.
- AWS organisations: for grouping accounts into OUs. Takes advantage of consolidated billing - which facilitates group discounts. Apply `service control policies` (SCPs) to set boundaries of permission for each account. `SCPs` are like IAM permission boundaries but at the org level!
- `SCPs` allow only what is at the intersection of IAM permissions and SCPs; like permission boundaries, `SCPs` do not grant permissions; they act as a filter.
- IAM layered defense: `SCP` <--> `IAM Permission Boundaries` <--> `IAM Policies`. Evaluation policy details: https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.htm
---

## Module #3 - Networking 1 ##
All about IP addressing, VPCs, network security.

### CIDR (Classless Inter-Domain Routing) ###
- CIDR specifies a range of IP addresses called a CIDR block.
- CIDR block numbers are same as subnet mask (use https://www.calculator.net/ip-subnet-calculator.html).

### VPC ###
- A logical isolation for your workloads; you can create any num of VPCs and by default VPCs do not communicate with each other.
- VPC apply to a region and will be used across AZs.
- Your VPC can have up to five CIDR blocks, and their address ranges cannot overlap.
- Identical CIDR blocks, e.g. 172.31.0.0/16 can be assigned to different VPCs since they are isolated but unique CIDR blocks need to be assigned when you connect VPCs together. Hence plan ahead early!!!

### VPC - Subnets ###
- A subnet resides within one AZ.
- All subnets created are private by default.
-  A public subnet must have an internet gateway and route to the gateway. You create and attach the internet gateway and route tables to make a subnet public in AWS.
-  If your subnet is associated with a route table that has a route to an internet gateway, it’s known as a public subnet.
- A subnet does not allow outbound traffic by default. Your VPC uses route tables to determine where to route traffic. To allow your VPC to route internet traffic, you create an outbound route in your route table with an internet gateway as a target, or destination.
- A public subnet holds resources that work with inbound and outbound traffic; needs: route table, internet gateway, elastic IP addresses.
- For example, Internet --> (Internet gateway <--> public route table <--> EC2 instance).
- Internet gateway is like reverse proxy (public to private brokering).
-  The VPC uses a main route table to connect the subnets to the internet gateway.
- Separate resources using public subnet (webservers) and private subnet (DB).
- Each VPC has a main route table that will contain destinations to different subnets (private and public).
- If you don't explicitly associate a subnet with a particular route table, the subnet is implicitly associated with and uses the main route table.
- A subnet can be associated with only one route table at a time, but you can associate multiple subnets with the same route table.
- Use custom route tables for each subnet to permit granular routing for destinations.
-You are limited to five Elastic IP addresses per VPC.
- NAT gateway needs to be deployed in the public subnet.
- Use NAT gateway to allow private subnet to have outbound traffic to the internet; maps private IP to public IP.
- You use NAT for IP address conservation and also for hiding source IP from private instances from internet services going through NAT --> Internet Gateway --> internet destination.
- For HA config, deploy VPC across AZs and configure identical subnets across the AZs, with a single Internet Gateway for inbound and outbound traffic management | NAT --> Internet Gateway for private outbound to internet destination | Internet Gateway --> Elastic LB for public inbound to private instances for load distribution.

### Elastic Network Interface ###
- This is like a virtual network card which can be "plug into" different EC2 instances while maintaining the same public and private IP addresses.
- You can attach a network interface in one subnet to an instance in another subnet in the same VPC. However, both the network interface and the instance must reside in the same Availability Zone. This restriction limits its use for disaster recovery (DR) scenarios, where you would want to redirect traffic to another Availability Zone.

### Network Access Control List (ACL) ###
- Network ACL is at subnet boundary; acts like a firewall as protection to subnets.
- Stateless: meaning each inbound / outbound traffic will apply the rules again.
- DENY all traffic is the default for any new subnet created in the ACLs.

### Security Groups ###
- Security Groups is for specifc resources, e.g EC2.
- Anything is allowed outbound but nothing is allowed inbound by default for security groups.
- Stateful: remembers what's granted without re-challenging/re-evaluation.
- Network ACL takes priority over Security Groups.
- Every VPC has a default security group that blocks all inbound and outbound for resources.
- Security groups allows chaining, whereby we can restrict access from web tier to only app tier, and app tier to only DB tier.
---

## Module #4 - Compute ##
All about EC2, instance storage, pricing, Lambda.

### Compute Services ###
- Conventional EC2s
- Containers: ECS, EKS
- Serverless: Lambdas, Fargate [serverless and containerised]
- Custom built-processors: trainiums, gravitons.
- Use `AWS Compute Optimizer` to apply and compare different instances types and costs impact, to choose the right sized compute.
- Types of tenancy: shared (default), dedicated instance (isolated hardware), dedicated host (controlled hardware - you own the entire hardware stack; most expensive).

### Storage for EC2 Instances ###
- Elastic Block Store: it is like a SAN storage to a EC2 | move EBS volumes between EC2 instances as needed | HDD and SSD types | works still like an internal disk - mounted disk.
- Instance Store Volumnes: local to instance and non-persistent (lost when EC2 is stopped) | used for temp data storage.

### Serverless ###
- Lambda: runs up to 15mins only | supports up to only 10GB of memory. 
- Very useful for IT automation.


