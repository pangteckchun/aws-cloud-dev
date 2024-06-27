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
---

## `Lab #1 - Using AWS CLI for s3` ##
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

## `Lab #2 - VPCs, subnets, EC2s` ##
- One VPC in a single region in one single AZ - 2 subnets [public and private] in the VPC.
- Public subnet - attach Internet Gateway and route table (making it publicly available) | Internet Gateway allowing internet traffic as default setting | attach NAT gateway, connect NAT Gateway to Internet Gateway in route table.
- Private subnet - attach route table, connect to NAT gateway in route table.
- Create EC2 in each public and private subnets.
- Try accessing private subnet EC2 - not allowed.
- Try accessing public subnet EC2 - allowed.
- Try accessing internet **from** private subnet EC2 - allowed.
---

## Module #5 - Storage ##
All about block. file and object storage.

### Fundamentals ###
- Block storage, i.e. `ebs` can only be attached to EC2 in the same AZ.
- File storage is like NAS, i.e. `efs`
- Object storage is `s3`.

### S3 ###
- Does not use traditional file and folder structure.
- SLA of 11 "9s" (99.999999999) of data durability in region, automatically replicated across AZs with minimal of 3 AZs redundancy.
- Individual objects must be <= 5TB.
- Amazon S3 maintains compliance programs, such as payment card industry data security standards (PCI-DSS), HIPAA/HITECH, FedRAMP, EU Data Protection Directive, and FISMA to meet regulatory requirements.
- Amazon S3 stores data as objects within buckets (folders); bucket names must be globally unique. Automatically encrypted (4 keys encryption schemes to choose from).
- Every object can be uniquely addressed through the combination of the web service 
endpoint, bucket name, key, and optionally, a version.
- `s3` blocks public access and is private by default. Additionally apply "Block public access" to further enforce access to buckets.
- Use bucket policy to apply ACL to `s3` without managing permissions in IAM.
- Create **Access Point** to allow get and put operations of objects to specific buckets (access points are attached to buckets) with certain prefix to control access coverage.
- 7 types of storage classes for `s3`: Standard, Standard-IA, One Zone-IA, Glacier Instant Retrieval, Glacier Flexible Retrieval, Glacier Deep Archive, Intelligent Tiering.
- When you use the Amazon S3 console to browse your storage, you incur charges for GET, LIST, and other requests that are made to facilitate browsing.
- Versioning in `s3` is not automatically turned on. This will increase storage and costs when turned on.
- Replication also incurs more costs as there are more PUT requests for your objects.
- Cross-region replication in `s3` has further bill implications as it incurs storage AND networking costs.
- Use `s3 Transfer Acceleration` which uses AWS global edge locations to facilitate fast access to `s3` bucket over an optimized path.
- `s3` can also send notifications for any object changes to AWS SNS, SQS or Lambda.

### EFS ###
- Serverless file storage which scales automatically without a need to specify storage size upfront.
- Middleground between `ebs` (local to EC2s) and `s3`. Used for mutiple instances access to a single storage location (EC2s, ECS, EKS, Fargate, Lambda) **within the same VPC**.
- Storage limit of 1.6PB.
- Storage classes of standard, one zone.
- Use FSx types (Windows, Lustre - Linux OSes, NetApp ONTAP, OpenZFS) to leverage familiar native file storage types for your applications.

### Comparison ###
- Access:  
-- `ebs` - direct attach to EC2  
-- `efs` - VPC (i.e. LAN)  
-- `s3` - Internet  

- Capacity:  
-- `ebs` - 64TB per disk  
-- `efs` - 1.6PB  
-- `s3` - Unlimited  

- Performance:  
-- `ebs` - 256K operations  
-- `efs` - 10K clients  
-- `s3` - No limit  

- Costs:  
-- `ebs` - $$$  
-- `efs` - $$  
-- `s3` - $   

### Data Migration Tools ###
- [online] Storage gateway applicance, data sync, transfer family
- Storage gateway types:  
-- S3 file gateway, FSx file gateway, tape (s3 objects) gateway, volume (block/ebs) gateway.  
-- Install on-premise and choose destinations for S3, FSx (windows only), S3 Glacier, EBS according to the types above respectively.
- Datasync:  
-- agent based and only for EFS and S3 contents but not for EBS volumes.
- [offline] snow family (offline - snowcone, snowball edge, snow mobile) - very large datasets and time critical to complete.
---

## Module #6 - Database Services ##
All about RDBMS, No-SQL, caching, db migration tools; AWS database-as-a-service offerings.

### Types of Database Services ###
- See  https://aws.amazon.com/products/databases/
- Amazon RDS (for PostgreSQL, MySQL, MariaDB, Oracle Database Standard Edition, and Microsoft SQL Server, including Amazon Aurora) - managed RDBMS.
- Amazon Aurora (for PostgreSQL, MySQL only) - managed RDBMS.

### Amazon RDS ###
- AWS managed (patching, backups, restoration); data encrypted at rest and in-transit, industry compliance, multi-AZ replication, auto-scaling with minimal downtime to your application.
- In multi-AZ setup, a standby DB instance is setup in the different AZ to replicate data over; not used for read for this standby instance; automatic failover.
- Can also create a **read replica** to load shed read operations from your primary DB; uses async replication for the read replica; have the ability to put this replica in a single AZ or across region.
- uses AWS KMS for keys to encrypt data at rest.

### Amazon Auora ###
- A MySQL and PostgreSQL compatible relational database built for the cloud.
- It is up to five times faster than standard MySQL databases and up to three times faster than standard PostgreSQL databases.
- Much better performance, availability & durability than RDS version of MySQL and PostgreSQL.
- Automatically 6 copies of data (6 instances of DB) across 3 AZs on setup, with a cluster volumes for high durability. None of the replicas are standby instances but read replicas sync **in real-time** and can takeover primary instance in real-time.

### Amazon DynamoDB ###
- Serverless, scalable key-value NoSQL DB.
- Optimized specifically for applications that require large data volume, low latency, and flexible data models.
- It can also manage many concurrent read and write operations while maintaining the accuracy of stored data.
- Read requests for up to 3 to 4KB per item; write request for up to 1KB per item.
- Manage DynamoDB capacity using on-demand or provisioned (if you already know your read/writes volume - but still autoscaled, within min and max limits you can set).
- DynamoDB consistency options: eventual consistent read (delays in data sync across AZs) and strongly consistent read (no delays but twice more expensive).
- Cross region replication (for global tables) is supported but in async replication.

### Data Caching ###
- Two common caching strategies include lazy loading and write through.
- Lazy loading - cache miss then retrieve from source database.
- Write through - persist to both cache and database.
- Evict cache using TTL, least recently used, least frequently used strategies.

### Amazon ElastiCache ###
- A web service that facilitates setting up, managing, and scaling a distributed in memory data store or cache environment in the cloud.
- Supports Redis and Memcached.

### Amazon DynamoDB Accelerator (DAX) ###
- DAX is a caching service compatible with DynamoDB that provides fast in memory performance for demanding applications.

### Amazon Database Migration Service ###
- Covers migration across different DB types, which include Oracle, PostgreSQL, SQL 
Server, Amazon Redshift, Aurora, MariaDB, and MySQL.
- Either the source or the target database (or both) must reside in Amazon RDS or on Amazon Aurora.
- Automatically handles formatting of source data for target DB.
- Use `AWS Schema Conversion Tool` (`AWS SCT`) to convert schema between heterogeneous DBs migration; The conversion includes views, stored procedures, and functions; it converts legacy Oracle and SQL Server functions to their equivalent AWS service, thus modernizing the applications at the same time of database migration.
- AWS SCT can also help migrate data from various data warehouses to Amazon Redshift by using built-in data migration agents.

## `Lab #3 - ALB & Aurora DB` ##
- ALB in public subnet --> target group (with rule) --> EC2s @ 2 AZs --> DB-security-group --> Aurora DB instance
- Access custom web app (on EC2s) using ALB public DNS name, put in DB credentials via settings and you can interact with the DB via the custom web app page.
---

## Module #7 - Monitoring & Scaling ##
All about monitoring, alarms & events, load balancing and auto-scaling.  
Alarm states include "OK", "ALARM", "INSUFFICIENT_DATA".

### Types of Logs ###
- `Amazon CloudWatch` Logs monitor, store, and access your log files from Amazon EC2 instances, AWS CloudTrail, Amazon Route 53, and other resources. 
- `AWS CloudTrail` provides event history of your account activity, including actions taken through the console, AWS SDK, command line interface (CLI), and AWS services. This event history simplifies security analysis, resource change tracking, and troubleshooting. CloudTrail facilitates governance, compliance, and operational and risk auditing. With CloudTrail, you can log, continuously monitor, and retain account activity that is related to actions across your AWS infrastructure. 
- `VPC Flow Logs` captures information about the IP traffic that goes to and from network interfaces in your virtual private cloud (VPC). 
- `Load balancing` provides access logs that capture detailed information aboutrequests that are sent to your load balancer. You can use custom logs from your applications.

### Amazon CloudWatch  ###
- Free and automatically given to you once you use any AWS services.
- Near real-time metrics and logs.
- Create alarms and send notifications & dashboarding.
- Amazon CloudWatch can load all the metrics in your account (both AWS resource metrics and application metrics that you provide) for search, graphing, and alarms.
- Metric data is kept for 15 months, so you can view both up-to-the-minute data and historical data.
- To graph metrics in the console, you can use CloudWatch Metrics Insights, a high-performance structured query language (SQL) query engine.
- Typically, logs --> [AWS] log stream for each source (AWS or on-premise) --> log groups (A log group shares the same retention, monitoring, and access control settings across all streams) --> matching phrases, keywords --> count & generate metrics.
- `Amazon EventBridge` is the preferred way to manage your events that are captured in CloudWatch.

### AWS CloudTrial  ###
- AWS CloudTrail provides insight into who did what and when by tracking user activity and API usage.
- These calls can be made through the console, AWS SDKs, CLI, and higher-level AWS services.
- You turn on CloudTrail on a per-Region basis.
- CloudTrail saves the logs in your designated `s3` bucket.

### VPC Flow Logs  ###
- Captures IP traffic information to and from VPC network interfaces.
- Flow logs are turned off by default.
- Flow logs are published to either Amazon S3 buckets or CloudWatch log groups.

### Elastic Load Balancer (ELB)  ###
- 3 types: Application LB (L7 apps traffic - http/s), Network LB (L3/4 network traffic - TCP, UDP), Gateway LB (L3 virtual applicance traffic).
-  Your load balancer can forward requests to one or more target groups based on rules and settings that you define.
- Each target group routes requests to one or more registered targets (for example,EC2 instances) by using the specified protocol and port number.

### Auto-Scaling ###
- AWS Auto Scaling provides application scaling for multiple resources such as Amazon EC2, Amazon DynamoDB,Amazon Aurora, and many more across multiple services in short intervals.
- `Amazon EC2 Auto Scaling` can launch or terminate instances as demand on your application increases or decreases.
- `Amazon EC2 Auto Scaling` integrates with ELB so that you can attach one or more load balancers to an existing Amazon EC2 Auto Scaling group.
- Specify what resources is needed using launch templates, as part of auto scaling group.
- Specify min and max resources in the auto scaling group.
- Specify scaling based on: health status, cloudwatch alarms, schedules (date/time), manually (you specify capacity on your scaling group yourself).

## `Lab 4 - Auto-scaling groups` ##
- [Security] Http (80) --> **ALB** --> secGrp-ALB -- allowed by --> secGrp-EC2s inbound --> **EC2s** --> secGrp-EC2s outbound --> secGrp-DB inbound --> **DB**.
- [Flow] **ALB** target-grp --> attach ALB to scaling-grp of EC2s by selecting the ALB's target-grp:  
-- target-grp specifies the EC2 instances as targets;  
-- combining with scaling-grp will allow auto-scaling to launch more EC2 instances when more traffic is experienced by ALB.  
- Create launch template (OS image, instance type etc) --> create auto-scaling group and attach launch template --> associate with ALB & its target group:  
-- Launch templates specify the EC2 details (OS, instance types etc) which will be launched wh  en scaling up more instances.
- Observed lost of web app functionality when one app instance is terminated; not seamless HA.
- Manually add reader to DB primary instance but need to manually note to use different AZ from primary instance.
- Manually failover by going to `databases`, select cluster and go a failover.
- Review via `events` as the failover is occurring. Notice that the Read replica instance is shutdown, promoted to the writer and then rebooted. When the reboot of the read replica is completed, then the inventory-primary is rebooted.
---

## Module #8 - Automation ##
All about CloudFormation, Elastic Beanstalk & others.

### AWS CloudFormation ###
- AWS IaC service with most granular control of what to provision.
- Will detect drfit and only apply changes that are different from running env.
- Author your CloudFormation template with any code editor and then check it in to a version control system such as GitHub or CodeCommit. Review files before deploying.
- Use nested stacks (for different tier of your solution) and use CloudFormation create the different layers for you. Stacks can have inter-dependency.
-  If a resource can't be created, CloudFormation rolls the stack back and deletes any resources that were created. If a resource can't be deleted, CloudFormation retains any remaining resources until the entire stack can be successful.

### AWS Elastic Beanstalk ###
- Elastic Beanstalk integrates with developer tools and provides a one-stop experience to manage the application lifecycle.
- Elastic Beanstalk provisions and manages application infrastructure to support your application for your application type.

### AWS Solutions Library ###
- AWS Solutions Library carries solutions that AWS and AWS Partners have built for a broad range of industry and technology use cases that solves common problems and build faster.
- These solutions include the tools that you need to get started quickly, such as CloudFormation templates, scripts, and reference architectures.

### AWS Cloud Development Kit (CDK) ###
- AWS CDK is an open-source software development framework to model and provision your application resources by using common programming languages such as Python, JavaScript, TypeScript, Java or C#.
- AWS CDK simplifies the creation and deployment of CloudFormation templates. It offers infrastructure components and groups of components that are preconfigured according to best practices. However, you can still customize your components and their settings.

### AWS Systems Manager ###
- View and control your infrastructure on AWS and can automate or schedule a variety of maintenance and deployment tasks.

### Amazon CodeWhisperer ###
- AI-assisted coding assistant for infra automation, resource creation (e.g. db creation); using comments to generate code for infra creation for you.
- AI security scanner for your CDK or infra related application codes.
---

## Module #9 - Containers ##
All about microservices, containers and container services.

### Microservices ###
- Read https://docs.aws.amazon.com/whitepapers/latest/microservices-on-aws/simple-microservices-architecture-on-aws.html

### Containers ###
- Containers share an operating system installed on the server and run as resource-isolated processes, which offers quick, reliable, and consistent deployments, regardless of the environment.
- You can use an EC2 and deploy Docker to manage containers but not scalable to only the size of the EC2 instance.

### Container Services ###
- Amazon ECR (registry) --pulled by--> EKS or ECS --deploy to--> Fargate (serverless containers runtime) or EC2. You can mix EKS-EC2 or EKS-Fargate or ECS-EC2 or ECS-Fargate.
- `Amazon Elastic Container Registry`(`Amazon ECR`) is a managed Docker container registry. An `Amazon ECR` private registry is provided to each AWS account.
- `EC2` hosting option: You can choose Amazon EC2 to launch containers on a variety of instance types. As demand changes, you can scale out and scale in the number of EC2 instances that are used to host containers. But you need to manage the EC2 compute | use `Fargate` to remove this overhead.
- `AWS Fargate` is a technology for `ECS` and `EKS` that you can use to run containers without having to manage servers or clusters. With Fargate, you no longer need to provision, configure, and scale clusters of VMs to run containers.
- `ECS` vs `EKS`: `ECS` is a more managed solution but proprietary to AWS. `EKS` is open-sourced aligned but needs more configuration to use but can run anywhere as long as Kubernetes is supported.
---

## Module #10 - Networking 2 ##
All about VPC endpoints, peering, hybrid networking, AWS Transit Gateway.

### VPC Endpoints ###
- Without VPC endpoints, a VPC requires an internet gateway and a NAT gateway, or a public IP address, to access serverless services outside the VPC.
- You **do not need an internet gateway, a NAT device, a VPN connection, or an AWS Direct Connect connection**. Instances in your VPC do not require public IP addresses to communicate with resources in the service; uses private IP only.
- Still protects private resources from internet traffic.
- EC2 @ private subnet --> `VPC endpoint` --> AWS services.
- 2 types: gateway and interface VPC endpoints.  
-- Interface VPC endpoint uses ENI (elastic network interface - virtual network card) to connect different services. Only need one Interface VPC endpoint "instance" to connect different internal services. No need to configure any new entry in route table. Need to specify which subnet the Interface VPC endpoint will reside and it will take on the IP subnet mask.  
-- Gateway endpoint only talks to S3 and DynamoDB. No charges for gateway endpoint for charges apply for data transfer and resource usage. Needs to configure routing table for the subnet to talk to each gateway endpoint. Each S3 or DynamoDB service needs a separate Gateway VPC endpoint. Gateway VPC endpoint is at VPC level and not tied to any subnet.  

### VPC Peering ###
- To allow VPCs to private route traffic.
- The maximum transmission unit (MTU) across a VPC peering connection is 1,500 bytes.
- 1-to-1 relationship between 2 VPCs.
- Need to configure VPC routing table on both VPCs to peer.
- IP spaces (CIDR ranges) between the 2 VPCs cannot ovelap; need unique IP address across the 2 VPCs for each resource.
- Free to use.
- VPC peering relationship is NOT transitive.
- Allows peering across regions too.
- Can become very messy if you need VPCs to communicate amongst each other and you have many VPCs to manage.

### AWS Site-to-Site VPN ###
- Connects on-premise to AWS cloud in encrypted comms, over public internet.
- Hardware config @ on-premise installation to speak to virtual private gateway (VPC endpoint type) @ AWS.
- No charges for data going into AWS; outgoing yes.
- No charges for inactive connections.

### AWS Direct Connect ###
- Fiber connection from your own DC to AWS DC, via a 3rd telco party.
- Not encrypted in transit by default but it is a private dedicated connection.
- A physical hub (direct connect location) owned by AWS is connected by the Telco from your DC; avoids disclosing the DC locations of AWS.
- Very high bandwidth but an expensive option.
- Need to pay local telco (bandwidth usage) and AWS (for port hours usage).
- You still pay for port hours even if there is no active connections / usage.

### AWS Transit Gateway ###
- A solution to reduce routing table maintenance.
- Acts like a hub-and-spoke and can connect up to 5,000 VPCs and on-premise environments, per gateway. Allows VPCs to comms with each other via the gateway.
- Routing through a transit gateway operates at Layer 3, where the packets are sent to a specific next-hop attachment based on their destination IP addresses; it scales elastically, based on traffic.
- Inter-region connections too but you will need other region to also use transit gateway and do peering. Transit Gateway has its own internal routing table to facilitate peering.
- Supports external site-to-site VPNs and Direct Connect into your network via the transit gateway.
- Allows you to control who can talk to who via the transit gateway. Can even deny inter-VPCs comms through config.
---

## Module #11 - Serverless ##
All about API gateway, SQS, SNS, Kinesis, Step Functions.

### Amazon API Gateway ###
- API Gateway handles all of the tasks involved in accepting and processing up to hundreds of thousands of concurrent API calls.
- These tasks include traffic management, authorization and access control, 
monitoring, and API version management.
- Built-in DDoS protection & allows you to do API throttling.
- Host and use multiple versions and stages of your APIs.
- Create and distribute API keys to developers.
- Use Signature Version 4 (SigV4) to authorize access to APIs.
- Use RESTful or WebSocket APIs.
- API Gateway also sends logs to Amazon CloudWatch. API Gateway can send logs to CloudWatch for each stage in your API or for each method. You can set the verbosity of the logging (Error or Info), and whether full request and response data should be logged.
- Metrics collected by API Gateway:  
-- Number of API calls  
-- Latency  
-- Integration latency  
-- HTTP 400 and 500 errors  

### Amazon Simple Queue Service (SQS) ###
- Amazon SQS is a fully managed service that requires no administrative overhead and little configuration.
- Charged by the num of times you poll SQS.
- Operates within a single, highly available AWS Region with multiple redundant Availability Zones.
-  Streaming Single Instruction Multiple Data (SIMD) Extensions (SSE) protects the contents of messages in SQS queues using keys managed in AWS KMS. SIMD Extensions encrypts messages as soon as Amazon SQS receives them. The messages are stored in encrypted form, and Amazon SQS decrypts messages only when they are sent to an authorized consumer.
- Once the maximum number of retries is reached, SQS can redirect the message to an Amazon SQS dead-letter queue where you can reprocess or debug it later.
- 2 types of SQS queue types:  
-- Standard queues support at-least-once message delivery and provide best-effort ordering. Messages are generally delivered in the same order in which they are sent. But distributed processing can mean they can be out of order. 
-- FIFO queues are designed to enhance messaging between applications when the order of operations and events is critical or where duplicates can't be tolerated. FIFO queues also provide exactly-once processing, but have a limited number of API calls per second.
- Dead-letter queues are supported out-of-the-box for failed messages; configurable (disabled / enabled).
- The consumer of a message from SQS needs to delete the message once it completes processing the message. If the consumer fails to delete the message before the visibility timeout expires, it becomes visible to other consumers and can be processed again.
- Configurable short polling & long polling; SQS does not notify consumers like Kafka. Consumers need to poll SQS. Short polling can mean message may not have arrive when polled. Long polling - SQS will not return a response to consumer until at least 1 message has arrived.
- SQS only supports one consumer per queue. SQS is like the old Queue type in JMS.

### Amazon Simple Notification Service (SNS) ###
- Amazon SNS is a web service that helps you to set up, operate, and send notifications from the cloud.
- You create a topic and control access to it by defining policies that determine which publishers and subscribers can communicate with the topic.
- Publisher-Subcriber model, i.e. 1-to-many.
- Use cases: CloudWatch alarms, email and SMS messages push, mobile app push notifications.
- Also supports standard and FIFO queue types.
- Typically used in conjunction with SQS to perform a fan-out architecture:  
-- SNS --to many--> SQS queues --to one--> consumer (1 SQS per consumer).  
- SNS is like the old Topic type in JMS but it does not have message persistency. Push mechanism as compared to SQS.

### Amazon Kinesis ###
- Streaming data processing: real-time data analytics by separate consumers, data firehose for mini-batch processing (near real-time) and store into backend, 
- Also supports Apache Flink and video stream processing.

### Amazon Step Functions ###
- Serverless orchestration tool.
- Step Functions use a JSON-based Amazon States Language, which contains a structure made of various states, tasks, choices, error handling, and more.
- Coordinates microservices using visual workflows.
- Provides simple error catching and logging if a step fails.
- Automatically initiates and tracks each step.
- Permits you to step through the functions of your application.
- 2 types of functions:  
-- Standard workflow type for long-running, durable, and auditable workflows.  
-- Express workflow type for high-volume, event-processing workloads such as IoT data ingestion, streaming data processing and transformation, and mobile application backends.  

## `Lab #5 - Serverless Build` ##
- `s3` --create events notification to SNS in S3--> `sns` (configure access policy to allow `s3` to trigger).
- `sns` configure subscriptions to separate `sqs` queues.
- `lambda` add trigger to each `sqs` queue and configure code that will run for each lambda trigger. One lambda config to each sqs queue.
- CloudWatch automatically monitors lambdas; view from **Monitor** tab within Lambda.
---

## Module #12 - Edge Services ##
All about Route 53, CloudFront, DDoS Protection, AWS Outposts.

### Edge Fundamentals ###
- Internet --> `Route 53` --> `AWS WAF` --> `Amazon CloudFront` (CDN) <--> [On-premise] `AWS Outposts`
- <-------- [Within AWS] protected by `AWS Shield` (DDoS protection)------>

### Route 53 ###
- Route 53 provides a DNS, domain name registration, and health checks.
- Resolves domain names to IP addresses.
- Registers or transfers a domain name.
- Routes requests based on latency, health checks and other criteria.
- Public DNS: route to internet-facing resources, resolve from the internet, use global routing policies.
- Private DNS: route to VPC resources, resolve from inside VPC.
- Different routing policies to work with: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html
- It is not a substitute for a load balancer, even though Route 53 can do health checks (failover routing, multivalue answer routing).

### Amazon CloudFront (global CDN service) ###
- Regional edge caches when you have content that is not accessed frequently enough to remain in an edge location.
- CloudFront supports real-time, bidirectional communication over the WebSocket protocol.
- Built-in CloudFront integrations with AWS Shield and AWS WAF.
- CloudFront speeds up the distribution of your content by routing each user request through the **AWS backbone network** to the edge location that can best serve your content.
- By requesting content from the Distribution domain value for the CloudFront configuration, you are verifying that the existing cache is working.

### AWS Shield ###
- DDoS is most common at the Network (Layer 3), Transport (Layer 4), Presentation (Layer 6), and Application (Layer 7) layers.
- Shield Standard (Layer 3 and 4) is out of the box once you have public address or when you use CloudFront, at no additional charge. Shield Advanced needs additional purchases (Layer 6 & 7).
- Best practices for DDoS resiliency: https://docs.aws.amazon.com/whitepapers/latest/aws-best-practices-ddos-resiliency/welcome.html.

### AWS WAF ###
- AWS WAF is a web application firewall that helps protect your web applications or APIs against common web exploits and bots.
- AWS WAF gives you control over how traffic reaches your applications. Create security rules that control bot traffic and block common attack patterns, such as SQL injection (SQLi) or cross-site scripting (XSS).
- You can also monitor HTTP(S) requests that are forwarded to your compatible AWS services.
- Logs are collected via CloudWatch Logs and metrics are collected via CloudWatch.

### AWS Firewall Manager ###
- AWS Firewall Manager simplifies the administration and maintenance tasks of your AWS WAF and Amazon VPC security groups.
- Set up your AWS WAF firewall rules, Shield protections, and Amazon VPC security groups once.
- There are three prerequisites for using it: activate AWS Organizations with full features, use AWS Config, and have an assigned user as the Firewall Manager administrator.

### AWS Outposts ###
- Extend the AWS Cloud to an on-premises data center.
- Outposts rack - from AWS in full hardware suite. Roll out by AWS.
- Outposts server - less space constraints; server delivered by AWS but installed on your own or vendor. Once connected to your network, AWS will remotely provision compute and storage resources.
- A virtual private cloud (VPC) spans all Availability Zones in its AWS Region. You can extend any VPC in the Region to your Outpost by adding an Outpost subnet.
- Outposts support multiple subnets.
- Each Outpost can support multiple VPCs that can have one or more Outpost subnets.

## `Lab #6 - CloudFront CDN Distribution` ##
- `CloudFront` --> [OAC config] --> `s3`
- Access `CloudFront` via its distributed domain name (from General config tab) in a browser and a simple web page is loaded displaying the information (instance id & zone) of the web server from which CloudFront retrieved the content.
- If a cache is missed, CloudFront will request from its configured origin which defines the location of the definitive, original version of the content that will be delivered through the CloudFront distribution. Usually this is the ALB DNS which fronts the EC2 content servers.
---

## Module #13 - Backup & Recovery ##
All about Disaster Planning, AWS Backup, Recovery Strategies.

### Disaster Planning ###
- **Note**: Fault tolerance is often confused with high availability, but fault tolerance refers to the built-in redundancy of an application's components to prevent service interruption. However, it is at a higher cost.
- AWS maintains a strict Region isolation policy so that any large-scale event in one Region will not impact any other Region.
- Recovery Point Objective(RPO) is the acceptable amount of data loss measured in time. For example, if a disaster occurs at 1:00 p.m. (13:00) and the RPO is 12 hours, the system should recover all data that was in the system before 1:00 a.m. (01:00) that day. Data loss will, at most, span 12 hours—between 1:00 p.m. and 1:00 a.m.
- Calculate RPO from backup frequency or interval; e.g. if you backup data every hour, your RPO is up to 1 hour.
- Recovery Time Objective (RTO) is the time it takes after a disruption to restore a business process to its service level, as defined by the operational level agreement (OLA). For example, if a disaster occurs at 1:00 p.m. (13:00) and the RTO is 1 hour, the DR process should restore the business process to the acceptable service level by 2:00 p.m. (14:00).
- Calculate RTO by how much time you need to restore services from backup / other means of services.
- When building your backup strategy, make a plan to duplicate your storage using a solution that meets your recovery needs:
-- `s3` - use cross-region replication.  
-- `s3 glacier` - stores in regional vaults; updated daily by AWS.  
-- `ebs` - point-in-time volume snapshots, then copy snapshots across regions and accounts.  
-- `snow family` - data transfer.  
-- `datasync` - sync to EFS.  
- For compute DR, AWS strongly recommends that you configure and identify your own AMIs so that they can launch as part of your recovery procedure. Your AMIs should be preconfigured with your operating system of choice, plus the appropriate pieces of the application stack.
- Network failover: leverage highly available Route 53 and ELB to detect failures and route across AZs and VPCs in failover mode.
- For `Amazon RDS`, achieve HA using:  
-- Create a Multi-AZ DB instance deployment, which creates a primary instance and a standby instance to provide failover support. However, the standby instance does not serve traffic.  
-- Create a Multi-AZ DB cluster deployment, which creates two standby instances that can also serve read traffic.  
-- Use a snapshot to create a Read Replica in a different Region. This replica can be promoted to primary in the event of disaster; promoting a Read Replica requires a reboot and has a higher RTO than failing over to a standby instance. 
-- Save a manual database (DB) snapshot or a DB cluster snapshot in a separate Region.  
-- Share a manual snapshot with up to 20 other AWS accounts.  
- Leverage `CloudFormation` stacks to restore and stand up your environment in the event of disaster.
- 

### AWS Backup ###
- `AWS Backup` is a fully managed backup service that helps you centralize and automate the backup of data across AWS services.
- Can back up databases such as DynamoDB tables, Amazon DocumentDB and Amazon Neptune graph databases, and Amazon RDS databases, including Amazon Aurora  database clusters.
- Can also back up Amazon EFS, Amazon S3, AWS Storage Gateway volumes, and all versions of Amazon FSx, including FSx for Lustre and FSx for Windows File Server.
- AWS Backup works with other AWS services to monitor its workloads, such as Amazon CloudWatch, Amazon EventBridge, AWS CloudTrail, and Amazon Simple Notification Service (Amazon SNS).

### Recovery Strategies ###
- Backup & restore: Use `s3` as your data backup location and restore it yourself, also using AMI for EC2 recreation and CloudFormation for networking redeployment.Longest RTO.
- Pilot light: replicate compute resources to another environnent but shutdown. Use DB (async) replication to backup data to another DB instance. Very much like hot primary - cold DR.
- Warm standby: similar to pilot light but compute resources are lower capacity. Leverage scaling group to scale up when site acts primary during failover. Like converting non-prod to a prod type of failover.
- Multi-site active/active: Hot primary and secondary DC. Always on compute and db. Traffic load balances across both DCs all the time as full site HA. Most expensive but least downtime and RPO-RTO.