# 🚀 AWS Custom VPC, Bastion Host, Private EC2, IAM Roles, S3 Endpoint & SSM

This project documents a hands-on AWS networking lab where I built a **production-style private network architecture** from scratch. The goal was not only to create AWS resources, but to understand how real cloud environments are designed, secured, accessed, and optimized.

The architecture includes a custom VPC, public and private subnets across two Availability Zones, an Internet Gateway, NAT Gateway, bastion host, private EC2 instance, IAM role-based S3 access, S3 Gateway VPC Endpoint, and AWS Systems Manager Session Manager.

This project represents the kind of foundation used before moving into more advanced AWS design structures such as Application Load Balancers (ALB), Auto Scaling Groups (ASG), RDS, Terraform, ECS, and EKS.

---

# 🧭 Architecture

```text
                                 ┌───────────────────────────┐
                                 │   Internet Gateway (IGW)  │
                                 └─────────────┬─────────────┘
                                               │
                ┌──────────────────────────────┼──────────────────────────────┐
                │                  VPC: 10.0.0.0/16                           │
                │                                                             │
                │   ┌──── ap-south-1a ────┐         ┌──── ap-south-1b ────┐   │
                │   │                     │         │                     │   │
                │   │  public-1           │         │  public-2           │   │
                │   │  10.0.1.0/24        │         │  10.0.2.0/24        │   │
                │   │  ┌──────────────┐   │         │                     │   │
                │   │  │  Bastion EC2 │   │         │   NAT Gateway       │   │
                │   │  │  + Public IP │   │         │   + Elastic IP      │   │
                │   │  └──────┬───────┘   │         │                     │   │
                │   │         │           │         │                     │   │
                │   │  private-1          │         │  private-2          │   │
                │   │  10.0.3.0/24        │         │  10.0.4.0/24        │   │
                │   │  ┌──────────────┐   │         │                     │   │
                │   │  │  Private EC2 │◄──┼─────────┼── (future EC2s)     │   │
                │   │  │  No public IP│   │         │                     │   │
                │   │  └──────────────┘   │         │                     │   │
                │   │                     │         │                     │   │
                │   │  rds-1              │         │  rds-2              │   │
                │   │  10.0.5.0/24        │         │  10.0.6.0/24        │   │
                │   └─────────────────────┘         └─────────────────────┘   │
                │                                                             │
                │           ┌─────────────────────────────────┐               │
                │           │  S3 Gateway VPC Endpoint        │               │
                │           │  (attached to private route tbl)│               │
                │           └─────────────────────────────────┘               │
                └─────────────────────────────────────────────────────────────┘
                                               │
                                               ▼
                                       ┌───────────────┐
                                       │  S3 Bucket    │
                                       │  (last week)  │
                                       └───────────────┘
```

---

# 🎯 Project Goals

The main goal of this project was to understand how AWS networking works beyond the default VPC. Instead of using pre-built AWS defaults, I manually created each networking component so I could understand how traffic actually flows.

By completing this project, the learning outcomes were to:

* Design a custom VPC using CIDR planning.
* Split a network into public, private, and database subnets.
* Use route tables to control traffic flow.
* Allow public resources to reach the internet through an Internet Gateway.
* Allow private resources to reach the internet through a NAT Gateway.
* Access private EC2 instances through a bastion host.
* Use IAM roles instead of hardcoded AWS credentials.
* Replace NAT-based S3 access with a free S3 Gateway VPC Endpoint.
* Access a private EC2 instance using AWS SSM Session Manager without SSH keys.

---

# 🧠 Core Design Principles

## 1. Isolation

Private resources should not be directly exposed to the internet. In this project, the private EC2 instance was launched without a public IP address. That means it could not be reached directly from my laptop or from the public internet.

This is important because production systems usually contain sensitive workloads such as APIs, backend services, and databases. These systems should only be reachable from trusted internal paths.

## 2. Controlled Access

Instead of connecting directly to private resources, access was controlled through a bastion host and later through AWS SSM Session Manager.

The bastion host acted as a controlled entry point into the private network. SSM improved this further by removing the need for SSH keys and open inbound SSH access.

## 3. Least Privilege

Resources should only have the access they need. IAM roles were used so that EC2 instances could access S3 without having to store access keys on the server. Later, the S3 permissions were restricted so the instance could access only the required bucket.

## 4. Cost Awareness

The NAT Gateway allowed private EC2 instances to reach the internet, but NAT Gateways cost money. To avoid sending S3 traffic through the NAT Gateway, I created an S3 Gateway Endpoint. This allowed private access to S3 without internet traffic and without NAT data processing charges.

---

# 📁 Repository Structure

```text
screenshots/
├── vpc/
├── subnet/
├── routing/
├── bastion/
├── private-ec2/
├── nat/
├── iam/
├── s3-endpoint/
└── ssm/

answers.md
reflection.md
gotchas.md
README.md
```

The screenshots are organized by the same workflow used to build the infrastructure. This makes it easier to follow the project from the first VPC step to the final SSM access step.

---

# 🧱 Step 1 — Creating the Custom VPC

The first step was creating a custom VPC named `bootcamp-vpc` with the CIDR block:

```text
10.0.0.0/16
```

A VPC is like a private data center network inside AWS. Every subnet, EC2 instance, route table, gateway, and endpoint in this project exists inside this VPC.

The `/16` CIDR range gives a large private IP address space. This is useful because a real production environment may grow over time and need additional subnets for application servers, databases, load balancers, containers, monitoring tools, and future services.

A smaller CIDR range can become a problem later because changing VPC CIDR planning after resources are already deployed can be difficult.

📸 Screenshot location:

```text
screenshots/vpc/
```

---

# 🌐 Step 2 — Creating Subnets Across Availability Zones

After creating the VPC, I divided the network into smaller subnets.

The subnets were created across two Availability Zones:

* `ap-south-1a`
* `ap-south-1b`

This matters because production systems should not depend on only one Availability Zone. If one AZ has an issue, resources in another AZ can continue running.

## Public Subnets

```text
public-1  → 10.0.1.0/24
public-2  → 10.0.2.0/24
```

Public subnets are used for resources that need a direct or indirect relationship with the internet. In this lab, the bastion host was placed in `public-1`, and the NAT Gateway was placed in `public-2`.

## Private Subnets

```text
private-1 → 10.0.3.0/24
private-2 → 10.0.4.0/24
```

Private subnets are used for internal application workloads. In this lab, the private EC2 instance was placed in `private-1` and had no public IP address.

## RDS Subnets

```text
rds-1 → 10.0.5.0/24
rds-2 → 10.0.6.0/24
```

RDS subnets were created to prepare for future database work. A production database should not live in the same network layer as public-facing resources.

📸 Screenshot location:

```text
screenshots/subnet/
```

---

# 🔀 Step 3 — Creating Route Tables

Route tables control where traffic goes. This is one of the most important parts of the lab because subnets are not truly public or private by name alone. They become public or private based on their routing.

## Public Route Table

The public route table was associated with the public subnets and contained this route:

```text
0.0.0.0/0 → Internet Gateway
```

This means any traffic going outside the VPC should go through the Internet Gateway.

A subnet with this route can support public resources, such as bastion hosts, NAT Gateways, and load balancers.

## Private Route Table

The private route table was associated with the private subnets. At first, it did not have a working internet route. Later, it was updated with:

```text
0.0.0.0/0 → NAT Gateway
```

This allowed the private EC2 instance to reach the internet for package updates, while still preventing inbound internet connections.

📸 Screenshot location:

```text
screenshots/routing/
```

---

# 🌍 Step 4 — Creating the Internet Gateway

An Internet Gateway was created and attached to the VPC.

The Internet Gateway is what allows resources in public subnets to communicate with the public internet. Without an Internet Gateway, even a public EC2 instance with a public IP would not be reachable.

The Internet Gateway does not automatically make every subnet public. The subnet must also have a route table entry pointing internet-bound traffic to the Internet Gateway.

---

# 🖥️ Step 5 — Launching the Bastion Host

A bastion host was launched in the public subnet.

The bastion host had:

* A public IP address
* SSH access
* A security group allowing SSH
* Placement inside the public subnet

The purpose of the bastion host is to act as a controlled jump server into the private network.

Instead of exposing the private EC2 instance directly to the internet, I connected first to the bastion and then from the bastion to the private instance.

The access pattern was:

```text
Laptop → Bastion Host → Private EC2
```

This pattern reduces exposure because only the bastion needs public access.

📸 Screenshot location:

```text
screenshots/bastion/
```

---

# 🔒 Step 6 — Launching the Private EC2 Instance

A private EC2 instance was launched inside `private-1`.

Important settings:

* No public IP
* Same VPC
* Private subnet
* Security group allowing SSH from the bastion path

The private EC2 instance was not reachable directly from the internet. This is exactly what we wanted.

This proved that the private subnet was isolated and that access had to go through the designed internal path.

📸 Screenshot location:

```text
screenshots/private-ec2/
```

---

# 🔐 Step 7 — SSH Access Through Bastion

To reach the private EC2, I first connected to the bastion host using SSH from my local machine.

Then I connected from the bastion host to the private EC2 using the private IP address of the private instance.

The private EC2 used an internal IP like:

```text
10.0.3.221
```

This confirmed that private IP communication inside the VPC was working.

This step showed the difference between:

* Public IP access from the internet
* Private IP access inside the VPC

---

# ❌ Step 8 — Testing Private EC2 Without Internet

Before creating the NAT Gateway, I tested internet access from the private EC2 by running:

```bash
sudo yum update -y
```

The command hung and failed to download packages.

This was expected because the private subnet did not yet have a path to the internet.

This test was important because it proved that the private EC2 was truly isolated.

📸 Screenshot location:

```text
screenshots/nat/
```

---

# 🌍 Step 9 — Creating NAT Gateway

A NAT Gateway was created in a public subnet and assigned an Elastic IP.

The private route table was updated with:

```text
0.0.0.0/0 → NAT Gateway
```

After this, the private EC2 could reach the internet, but the internet still could not initiate connections back to it.

This is the key security benefit of NAT:

```text
Private EC2 → Internet allowed
Internet → Private EC2 blocked
```

After adding NAT, I ran:

```bash
sudo yum update -y
```

This time it completed successfully.

📸 Screenshot location:

```text
screenshots/nat/
```

---

# 🔑 Step 10 — Attaching IAM Role for S3 Access

An IAM role was created and attached to the EC2 instances.

The purpose of the role was to allow EC2 to access S3 securely without storing access keys on the server.

This is much safer than running `aws configure` on the EC2 instance because hardcoded access keys can be stolen, leaked, or accidentally committed to GitHub.

With IAM roles, AWS provides temporary credentials automatically.

📸 Screenshot location:

```text
screenshots/iam/
```

---

# 📦 Step 11 — Testing S3 Access Through NAT

After attaching the IAM role, I tested S3 access from the private EC2.

The command worked while NAT existed because the traffic path was:

```text
Private EC2 → NAT Gateway → Public S3 endpoint
```

This proved two things:

1. The IAM role permissions worked.
2. The private EC2 could reach S3 through NAT.

However, this is not the most cost-efficient design because NAT Gateway data processing costs money.

---

# 🔒 Step 12 — Applying Least Privilege IAM

The broad S3 permissions were replaced with a more specific policy that allowed access only to the required S3 bucket.

This follows the principle of least privilege.

The EC2 instance should not be able to access every bucket in the AWS account if it only needs one bucket.

This is important in production because overly broad permissions increase risk.

---

# 💸 Step 13 — Removing NAT Gateway for S3 Optimization

To prove that S3 access depended on NAT, the NAT Gateway was deleted and the Elastic IP was released.

After NAT was removed, S3 access from the private EC2 failed or hung.

This proved the private instance no longer had a general internet path.

It also showed why relying on NAT for AWS service access can become expensive and unnecessary.

---

# 🔗 Step 14 — Creating S3 Gateway VPC Endpoint

An S3 Gateway VPC Endpoint was created and attached to the private route table.

This allowed the private EC2 to access S3 without NAT and without internet access.

The new traffic path became:

```text
Private EC2 → S3 Gateway Endpoint → S3
```

This is better because:

* Traffic stays on AWS internal networking
* NAT Gateway is no longer required for S3
* Costs are reduced
* Security is improved

📸 Screenshot location:

```text
screenshots/s3-endpoint/
```

---

# ⚠️ Step 15 — Troubleshooting Endpoint DNS Issue

During endpoint creation, an error occurred related to DNS settings.

The issue was that the VPC needed DNS support and DNS hostnames enabled.

This matters because AWS services depend on DNS names such as S3 and SSM endpoints. If DNS is disabled, the instance cannot properly resolve AWS service endpoints.

The fix was to enable:

* DNS resolution
* DNS hostnames

This was an important troubleshooting lesson because many AWS networking problems are actually DNS problems.

📸 Screenshot location:

```text
screenshots/s3-endpoint/
```

---

# 🔐 Step 16 — AWS Systems Manager Session Manager

Finally, AWS SSM Session Manager was used to access the private EC2 instance without SSH.

This is a more modern access pattern than using a bastion host.

SSM access required:

* IAM role with SSM permissions
* SSM Agent on the instance
* Network path to SSM services
* SSM-related VPC endpoints

The final access pattern became:

```text
Laptop → AWS SSM → Private EC2
```

This is better than SSH because:

* No SSH key files are needed
* No inbound port 22 is required
* Sessions can be audited
* Access is controlled through IAM

📸 Screenshot location:

```text
screenshots/ssm/
```

---

# 🔄 Final Traffic Flows

## Admin access using bastion

```text
Laptop → Bastion Host → Private EC2
```

## Admin access using SSM

```text
Laptop → AWS SSM → Private EC2
```

## Internet access before NAT removal

```text
Private EC2 → NAT Gateway → Internet
```

## S3 access after endpoint

```text
Private EC2 → S3 Gateway Endpoint → S3
```

---

# 🛡️ Security Lessons Learned

This project demonstrated several important cloud security lessons.

Private EC2 instances should not be directly exposed to the internet. IAM roles should be used instead of access keys. S3 access should be limited to only the required bucket. NAT should be used carefully because it costs money. SSM can reduce or remove the need for bastion hosts and SSH keys.

---

# 🧠 Troubleshooting Lessons Learned

During this lab, several real-world issues appeared:

* SSH failed when the wrong IP path was used.
* Private EC2 could not reach the internet before NAT.
* S3 access failed when the bucket region did not match the endpoint region.
* Endpoint creation failed when VPC DNS settings were disabled.
* AWS CLI required local credentials and the Session Manager plugin.

These issues were valuable because real DevOps work is mostly troubleshooting, verifying, and understanding why systems fail.

---

# ✅ What This Project Proves

This project proves practical understanding of:

* VPC design
* CIDR planning
* Subnet separation
* Internet Gateway
* NAT Gateway
* Bastion host access
* Private EC2 networking
* IAM roles
* Least privilege permissions
* S3 Gateway Endpoint
* SSM Session Manager
* Cloud troubleshooting

---

# 🚀 Future Improvements

I believe this architecture can be improved further by:

* Rebuilding the setup with Terraform
* Replacing bastion access fully with SSM
* Adding an Application Load Balancer
* Adding Auto Scaling Groups
* Moving the database layer to Amazon RDS
* Adding CloudWatch logs and VPC Flow Logs
* Adding stricter security group rules

---

# 🧠 Final Reflection

This was not just a click-through lab. It was a full AWS networking exercise that showed how production environments are structured.

The biggest lesson is that cloud networking is about controlling paths:

* Who can enter?
* Who can leave?
* Which services can be accessed privately?
* Which permissions are actually required?

Understanding these questions is what turns basic AWS usage into real DevOps engineering.

---

## 👤 Author

Built as part of hands-on DevOps and AWS networking practice. 
