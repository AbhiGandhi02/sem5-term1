# AWS: Cloud Basics, Core Services & VPC (with Terraform)

> **DevOps – Lecture 18 notes.** Anything marked **➕ Extra** was not covered in class. It's there because it fills a gap or comes up often in labs and interviews.
>
> **Builds on:** [Lecture 3 – Networking](03-Networking.md) (IP, CIDR/subnetting, NAT, firewalls: a VPC is all of that in the cloud) and [Lecture 17 – Terraform](17-Terraform-IaC.md) (we build the VPC as code).

---

## Table of Contents
1. [Cloud Basics: IaaS, PaaS, SaaS](#1-cloud-basics-iaas-paas-saas)
2. [Cloud Providers: AWS, Azure, GCP](#2-cloud-providers-aws-azure-gcp)
3. [➕ Regions & Availability Zones](#3--regions--availability-zones)
4. [Core AWS Services](#4-core-aws-services)
5. [VPC: Your Private Network in AWS](#5-vpc-your-private-network-in-aws)
6. [VPC Components](#6-vpc-components)
7. [Public vs. Private Subnets](#7-public-vs-private-subnets)
8. [Route Tables](#8-route-tables)
9. [Internet Gateway vs. NAT Gateway](#9-internet-gateway-vs-nat-gateway)
10. [Security Groups vs. NACLs](#10-security-groups-vs-nacls)
11. [➕ Bastion Hosts](#11--bastion-hosts)
12. [Hands-on: Build a VPC with Terraform](#12-hands-on-build-a-vpc-with-terraform)
13. [Quick Revision Cheat Sheet](#13-quick-revision-cheat-sheet)
14. [Likely Exam / Interview Questions](#14-likely-exam--interview-questions)

---

## 1. Cloud Basics: IaaS, PaaS, SaaS

**Cloud computing** = renting computing resources (servers, storage, databases) **over the internet, paying for what you use**, instead of buying and running your own hardware.

| Model | You manage | Provider manages | Example |
|-------|-----------|------------------|---------|
| **On-premises** | Everything | Nothing | Your own data centre |
| **IaaS**: Infrastructure as a Service | OS, runtime, app, data | Hardware, network, virtualisation | **AWS EC2**, Azure VMs |
| **PaaS**: Platform as a Service | **Just your app and data** | + OS, runtime, scaling | **AWS Elastic Beanstalk**, Heroku, Google App Engine |
| **SaaS**: Software as a Service | Only how you use it | **Everything** | **Gmail**, Slack, Salesforce |

```
                On-prem    IaaS      PaaS      SaaS
 Application      YOU      YOU       YOU       ☁️
 Data             YOU      YOU       YOU       ☁️
 Runtime          YOU      YOU       ☁️        ☁️
 OS               YOU      YOU       ☁️        ☁️
 Virtualisation   YOU      ☁️        ☁️        ☁️
 Servers/Network  YOU      ☁️        ☁️        ☁️
```

> 💡 **Simple words mein (pizza analogy):** On-prem = ghar pe khud pizza banao (sab tumhara). IaaS = kitchen kiraye pe lo, pizza khud banao. PaaS = pizza order karo, ghar pe khao (table, plates tumhari). SaaS = restaurant jaake khao, sab unka.
>
> ➕ **Shared responsibility model:** AWS secures the cloud itself (data centres, hardware). **You** secure what you put **in** the cloud (IAM users, security groups, your data, OS patches on EC2).

---

## 2. Cloud Providers: AWS, Azure, GCP

| Provider | Strength | Equivalent of EC2 / S3 |
|----------|----------|------------------------|
| **AWS** (Amazon) | Largest, the widest range of services, most widely used | EC2 / S3 |
| **Azure** (Microsoft) | Integrates with Microsoft products (Windows Server, Active Directory, Office 365) | Virtual Machines / Blob Storage |
| **GCP** (Google) | Data analytics, ML/AI, Kubernetes (GKE: Google created K8s) | Compute Engine / Cloud Storage |

The concepts carry over almost one-to-one, which is why Terraform's multi-cloud support matters ([L17 §2](17-Terraform-IaC.md#2-why-terraform)).

---

## 3. ➕ Regions & Availability Zones

You need these to build a VPC.

| Term | Meaning | Example |
|------|---------|---------|
| **Region** | A geographic area with its own set of data centres | `us-east-1` (N. Virginia), `ap-south-1` (Mumbai) |
| **Availability Zone (AZ)** | One or more **isolated data centres** inside a region, with separate power and network | `ap-south-1a`, `ap-south-1b` |

- A **VPC belongs to one region** and spans **all its AZs**.
- A **subnet lives in exactly one AZ**.
- **High availability** = put copies of your app in subnets in **2+ AZs**, so one data centre failing doesn't take you down.

---

## 4. Core AWS Services

### EC2: Elastic Compute Cloud (IaaS)
**Virtual servers** in the cloud. You pick the OS and size, and pay per second/hour.

| Concept | Meaning |
|---------|---------|
| **AMI** | The OS image to start from (Amazon Linux, Ubuntu …) |
| **Instance type** | Size: `t3.micro` (2 vCPU, 1 GB), `m5.large` … |
| **Key pair** | SSH key to log in (`ssh -i key.pem ec2-user@<ip>`, [L1](01-Linux-Permissions-and-Commands.md)) |
| **Security group** | The instance's firewall (§10) |
| **EBS** ➕ | The instance's disk (block storage), survives stop/start |
| **Elastic** | Scale up/down quickly. ➕ Auto Scaling Groups add/remove instances automatically |

**Use for:** hosting web apps and APIs, running backend servers, CI runners, Kubernetes worker nodes.
➕ **Pricing:** On-Demand (flexible) · Reserved / Savings Plans (cheaper for 1–3 years) · Spot (up to ~90% cheaper, can be interrupted).

### S3: Simple Storage Service
**Object storage**: store **any file** (images, video, backups, logs, static websites, Terraform state) as **objects** in **buckets**. Practically unlimited and very durable (11 nines).

**Storage classes**: trade-offs between cost, retrieval speed and how often you access the data:
| Class | For |
|-------|-----|
| **S3 Standard** | Frequently accessed data (default) |
| **S3 Intelligent-Tiering** | Unknown access patterns, moves objects automatically |
| **S3 Standard-IA** (Infrequent Access) | Rarely read, but needed fast when read |
| **S3 One Zone-IA** | Same, stored in one AZ only (cheaper, less resilient) |
| **S3 Glacier** (Instant / Flexible / Deep Archive) | Archives and backups, cheapest. Retrieval from milliseconds to hours. |

> ⚠️ **Correction:** the class notes mention "Standard and **Premium**" tiers. S3 has **no "Premium" class**. Use the table above. (➕ The fastest option is **S3 Express One Zone**.) Also, S3 isn't only for media: it stores **any** kind of file.

### DynamoDB: NoSQL database
- **Fully managed, non-relational** database. Data is stored as **items** (key-value / JSON-like documents) looked up by a **primary key**.
- **Single-digit-millisecond** reads at any scale, **no fixed schema** (each item can have different attributes).

| | **DynamoDB** | **RDS** |
|-|--------------|---------|
| Type | **NoSQL** (key-value/document) | **Relational SQL** (MySQL, PostgreSQL …) |
| Schema | Flexible | Fixed tables and columns |
| Queries | By key (fast, simple) | Rich SQL, JOINs |
| Scaling | Automatic, near-unlimited | Vertical, plus read replicas |
| Use for | Sessions, carts, leaderboards, IoT, high-traffic lookups | Orders, finance, data with relationships |

### IAM: Identity and Access Management
Controls **who** can do **what** on **which** AWS resources.

| Concept | Meaning |
|---------|---------|
| **Root user** | The account owner, with unlimited power. **Enable MFA, then don't use it day to day.** |
| **User** | A person or app with long-term credentials (password / access keys) |
| **Group** | A set of users that share permissions (`developers`, `admins`) |
| **Role** | Permissions **assumed temporarily** by a service or user, e.g. an **EC2 instance role**, so apps on EC2 get credentials **without stored keys** |
| **Policy** | A JSON document listing allowed/denied actions |

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::aarham-demo-bucket/*"
  }]
}
```
> **Least privilege:** give only the permissions needed, nothing more. By default IAM **denies everything**, and an explicit **Deny** always beats an Allow.

---

## 5. VPC: Your Private Network in AWS

A **VPC (Virtual Private Cloud)** is a **logically isolated private network** in AWS that **you** design: your own IP range, subnets, routing and firewalls. Every EC2 instance, database and load balancer lives inside a VPC.

> 💡 **Simple words mein:** AWS = ek bada shehar. VPC = usme tumhari apni gated society: apni boundary wall (CIDR), andar alag blocks (subnets), main gate (Internet Gateway), society ke andar ke raste (route tables), aur har ghar pe guard (security group).

```
 Region: ap-south-1                       VPC 10.0.0.0/16
 ┌─────────────────────────────────────────────────────────────────────────────────┐
 │                          Internet Gateway (IGW) ◀──────────────▶ 🌐 Internet    │
 │                                   ▲                                             │
 │  ┌──── AZ ap-south-1a ─────────────┼──────────┐                                 │
 │  │  PUBLIC subnet 10.0.1.0/24      │          │  route table: 0.0.0.0/0 → IGW   │
 │  │   [Web server / LB]  [NAT Gateway] ◀─┐     │                                 │
 │  │                                       │    │                                 │
 │  │  PRIVATE subnet 10.0.2.0/24           │    │  route table: 0.0.0.0/0 → NAT   │
 │  │   [App server]  [Database]  ──────────┘    │  (outbound only)                │
 │  └────────────────────────────────────────────┘                                 │
 │   every resource has a Security Group · every subnet has a NACL                 │
 └─────────────────────────────────────────────────────────────────────────────────┘
```

**The VPC CIDR:** pick a **private** range ([L3 §2](03-Networking.md#-extra-public-vs-private-ips)), usually `/16` (65,536 addresses), then split it into `/24` subnets ([L3 §4](03-Networking.md#4-subnetting--cidr)).

> ➕ Every AWS account has a **default VPC** in each region (`172.31.0.0/16`, public subnets only). It's handy for quick tests, but real projects build their own.

---

## 6. VPC Components

| Component | Job | Analogy |
|-----------|-----|---------|
| **VPC** | Your isolated network with a CIDR range | The gated society |
| **Subnet** | A slice of the VPC's IP range **in one AZ**, public or private | A block/tower inside it |
| **Route table** | Rules for **where traffic goes** (by destination IP) | Signboards on the roads |
| **Internet Gateway (IGW)** | Connects the VPC to the internet, **both ways** | The main gate |
| **NAT Gateway** | Lets **private** subnets reach the internet **outbound only** | A post office that sends your letters out with its own return address |
| **Security Group (SG)** | Firewall **per resource** (instance level) | The guard at each flat's door |
| **Network ACL (NACL)** | Firewall **per subnet** | The guard at each tower's entrance |

---

## 7. Public vs. Private Subnets

| | Public subnet | Private subnet |
|-|---------------|----------------|
| Route table has `0.0.0.0/0 → IGW` | ✅ **Yes**: this is what makes it public | ❌ No (goes to the NAT instead, or nowhere) |
| Instances get public IPs | Usually (`map_public_ip_on_launch = true`) | No |
| Reachable **from** the internet | ✅ (if the security group allows) | ❌ Never directly |
| Can reach the internet | ✅ Directly | ✅ Only through a NAT Gateway (outbound) |
| Put here | Load balancers, web servers, NAT Gateway, bastion host | **App servers, databases**, internal services |

> ⚠️ **Key point:** a subnet isn't "public" because of its name or `map_public_ip_on_launch`. It's public **only if its route table sends `0.0.0.0/0` to an Internet Gateway.** The class Terraform snippet creates the IGW but **no route table**, so its "public" subnet **can't actually reach the internet**. §12 fixes this.

**➕ AWS reserves 5 IPs in every subnet** (network address, VPC router, DNS, future use, broadcast). So a `/24` has **251** usable addresses, not 254 ([L3 cheat sheet: "AWS: −5"](03-Networking.md#15-quick-revision-cheat-sheet)).

---

## 8. Route Tables

A **route table** is a list of rules: **"traffic for destination X → send to target Y"**. Every subnet is associated with **exactly one** route table, and the **most specific match wins**.

**Public route table:**
| Destination | Target | Meaning |
|-------------|--------|---------|
| `10.0.0.0/16` | `local` | Traffic inside the VPC stays inside (automatic, can't be removed) |
| `0.0.0.0/0` | `igw-xxxx` | Everything else → the internet |

**Private route table:**
| Destination | Target | Meaning |
|-------------|--------|---------|
| `10.0.0.0/16` | `local` | Inside the VPC |
| `0.0.0.0/0` | `nat-xxxx` | Everything else → NAT Gateway → internet (outbound only) |

> ⚠️ **Correction to the class notes:** they say route tables can "direct insecure incoming traffic to a bastion host". Route tables **don't inspect or filter** traffic. They only choose the **next hop by destination IP**. **Allowing or blocking** traffic is done by **security groups and NACLs**. A **bastion host** is a separate idea (§11): a hardened server you SSH through to reach private instances.

---

## 9. Internet Gateway vs. NAT Gateway

| | Internet Gateway (IGW) | NAT Gateway |
|-|------------------------|-------------|
| Direction | **In and out** | **Out only**: private instances start the connection, replies come back |
| Used by | Public subnets | Private subnets |
| Lives | Attached to the **VPC** (one per VPC) | Inside a **public subnet**, with an **Elastic IP** |
| Cost | Free | ⚠️ **Hourly charge + per-GB charge** |
| Why | Web servers must be reachable | Private servers still need `yum update`, `pip install`, external APIs |

It's the same **NAT** idea from [L3 §6](03-Networking.md#6-nat-sharing-one-public-ip): many private IPs share one public IP for outgoing traffic.

```
 Private EC2 10.0.2.15 ──▶ private route table (0.0.0.0/0 → NAT) ──▶ NAT GW (public subnet, EIP 13.x.x.x)
                       ──▶ public route table (0.0.0.0/0 → IGW) ──▶ Internet
 Internet ──✖──▶ 10.0.2.15   (no inbound path: that's the point)
```

> 💡 **Simple words mein:** IGW = society ka main gate, log aa bhi sakte hain, ja bhi sakte hain. NAT = sirf "exit" wala gate. Andar wale bahar ja sakte hain, bahar wale andar nahi aa sakte.

---

## 10. Security Groups vs. NACLs

| | **Security Group** | **Network ACL** |
|-|--------------------|-----------------|
| Applies to | **Instance / ENI** level (EC2, RDS, LB) | **Subnet** level (every resource in it) |
| Rules | **Allow only** (no deny rules) | **Allow AND deny** |
| State | **Stateful**: if a request is allowed in, the reply is automatically allowed out | **Stateless**: return traffic needs its own rule (ephemeral ports 1024–65535) |
| Evaluation | **All** rules are checked together | Rules checked **in number order**, first match wins |
| Default | New SG: **deny all inbound, allow all outbound** | Default NACL: allow all in and out |
| Used for | **Main firewall**, used every day | An extra layer: block a bad IP range for a whole subnet |

> ⚠️ **Correction to the class notes:** they say security groups define "allowed **and denied**" traffic. Security groups have **only allow rules**. Anything not allowed is denied. **Explicit deny rules exist only in NACLs.**

**Typical security groups:**
```
 web-sg:  inbound 80, 443 from 0.0.0.0/0      · inbound 22 from <your-IP>/32 only
 app-sg:  inbound 8080 from web-sg            ← refers to ANOTHER SG, not an IP ✅
 db-sg:   inbound 5432 from app-sg only       ← the DB is reachable only by the app tier
```
> ➕ **Referencing another security group** instead of IP ranges is the AWS best practice: "anything in `app-sg` may reach the DB", even when instance IPs change. The same firewall idea as [L3 §12](03-Networking.md#12-network-security-basics).

**Traffic check order (inbound to an EC2 in a subnet):**
```
 Internet → IGW → route table → NACL (subnet, stateless) → Security Group (instance, stateful) → EC2
```

---

## 11. ➕ Bastion Hosts

Private-subnet instances have **no public IP**. How do admins SSH into them?

**A bastion host (jump box)** = a small, hardened EC2 instance in the **public** subnet. You SSH **to it**, then **from it** into private instances.
```bash
ssh -J ec2-user@<bastion-public-ip> ec2-user@10.0.2.15     # -J = jump through the bastion
```
- Bastion SG: allow port 22 **only from your IP**.
- Private instance SG: allow port 22 **only from the bastion's SG**.

➕ **Modern alternative:** **AWS Systems Manager Session Manager** gives shell access with **no bastion, no open port 22 and no SSH keys**, controlled through IAM.

---

## 12. Hands-on: Build a VPC with Terraform

Builds the full picture from §5: a VPC, a public and a private subnet, an IGW, **route tables with associations** (missing in class), an optional NAT Gateway, and a web security group. Uses the Terraform workflow from [L17](17-Terraform-IaC.md#11-the-core-workflow-init--fmt--validate--plan--apply--destroy).

### `providers.tf`
```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

### `variables.tf`
```hcl
variable "aws_region" {
  type    = string
  default = "ap-south-1"
}

variable "my_ip_cidr" {
  description = "Your public IP for SSH, e.g. 49.36.10.20/32 (curl ifconfig.me)"
  type        = string
}

variable "create_nat_gateway" {
  description = "NAT Gateway costs money per hour; keep false unless you need it"
  type        = bool
  default     = false
}
```

### `main.tf`
```hcl
# ---------- VPC ----------
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true                      # ➕ instances get DNS names
  tags                 = { Name = "devops-vpc" }
}

# ---------- Subnets (from class, + AZ and tags) ----------
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "${var.aws_region}a"
  map_public_ip_on_launch = true
  tags                    = { Name = "public-subnet" }
}

resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "${var.aws_region}a"
  tags              = { Name = "private-subnet" }
}

# ---------- Internet Gateway (from class) ----------
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "devops-igw" }
}

# ---------- Public route table: THIS is what makes the subnet public ----------
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = { Name = "public-rt" }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# ---------- NAT Gateway (optional, costs money) ----------
resource "aws_eip" "nat" {
  count  = var.create_nat_gateway ? 1 : 0          # count = 0 → not created
  domain = "vpc"
}

resource "aws_nat_gateway" "nat" {
  count         = var.create_nat_gateway ? 1 : 0
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public.id             # NAT lives in the PUBLIC subnet
  depends_on    = [aws_internet_gateway.igw]
  tags          = { Name = "devops-nat" }
}

# ---------- Private route table ----------
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "private-rt" }
}

resource "aws_route" "private_nat" {
  count                  = var.create_nat_gateway ? 1 : 0
  route_table_id         = aws_route_table.private.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.nat[0].id
}

resource "aws_route_table_association" "private" {
  subnet_id      = aws_subnet.private.id
  route_table_id = aws_route_table.private.id
}

# ---------- Security group for a web server ----------
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    description = "HTTP from anywhere"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "SSH only from my IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.my_ip_cidr]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"                             # all protocols
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# ---------- ➕ An EC2 web server in the public subnet ----------
data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.al2023.id
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]
  user_data              = <<-EOF
    #!/bin/bash
    dnf install -y nginx
    echo "<h1>Hello from my Terraform VPC</h1>" > /usr/share/nginx/html/index.html
    systemctl enable --now nginx
  EOF
  tags = { Name = "web-server" }
}
```

### `outputs.tf`
```hcl
output "vpc_id"            { value = aws_vpc.main.id }
output "public_subnet_id"  { value = aws_subnet.public.id }
output "private_subnet_id" { value = aws_subnet.private.id }
output "web_public_ip"     { value = aws_instance.web.public_ip }
```

### Run it
```bash
terraform init
terraform fmt && terraform validate
terraform plan  -var="my_ip_cidr=$(curl -s ifconfig.me)/32"
terraform apply -var="my_ip_cidr=$(curl -s ifconfig.me)/32"
# Outputs: web_public_ip = "13.233.x.x"

curl http://$(terraform output -raw web_public_ip)     # <h1>Hello from my Terraform VPC</h1> ✅

# Verify with the CLI
aws ec2 describe-vpcs --filters Name=tag:Name,Values=devops-vpc
aws ec2 describe-route-tables --filters Name=vpc-id,Values=$(terraform output -raw vpc_id)

# Clean up
terraform destroy -var="my_ip_cidr=$(curl -s ifconfig.me)/32"
```

> ⚠️ **Cost:** the VPC, subnets, route tables, IGW and security groups are **free**. The **t3.micro** EC2 is free-tier eligible on new accounts. A **NAT Gateway is NOT free** (hourly + data charges), so it's behind `create_nat_gateway = false` by default. **Always `terraform destroy` after a lab.**
>
> ➕ **Experiment:** delete the `aws_route_table_association.public` block and `apply` again. The `curl` stops working even though the IGW still exists. That proves the **route table** is what makes a subnet public.
>
> ➕ Real projects often use the community module **`terraform-aws-modules/vpc/aws`**, which builds all of this (plus multi-AZ) in ~15 lines ([L17 §13](17-Terraform-IaC.md#13--modules-reusable-infrastructure)).

---

## 13. Quick Revision Cheat Sheet

```
CLOUD MODELS  IaaS (EC2: you manage OS+) · PaaS (Elastic Beanstalk: just your app) · SaaS (Gmail: just use it)
              shared responsibility: AWS = security OF the cloud · you = security IN the cloud ➕
PROVIDERS     AWS (widest) · Azure (Microsoft ecosystem) · GCP (data/ML, GKE)
REGION / AZ ➕ region = geography · AZ = isolated data centre(s) · VPC = 1 region · subnet = 1 AZ
EC2           virtual servers · AMI · instance type · key pair · SG · EBS · On-Demand / Reserved / Spot ➕
S3            object storage for ANY file · buckets · Standard · Intelligent-Tiering · Standard-IA
              One Zone-IA · Glacier (Instant/Flexible/Deep Archive) · NO "Premium" class
DYNAMODB      managed NoSQL key-value/document · ms latency · flexible schema · vs RDS = relational SQL
IAM           users · groups · roles (temporary, EC2 instance role) · JSON policies · least privilege
              root + MFA, don't use daily · default deny · explicit Deny wins
VPC           isolated private network · private CIDR (10.0.0.0/16) · split into /24 subnets
PUBLIC SUBNET route table 0.0.0.0/0 → IGW (this is what makes it public) · LB, web, NAT, bastion
PRIVATE       no IGW route · 0.0.0.0/0 → NAT for outbound · app servers, DBs
ROUTE TABLE   destination → target · local route always · most specific wins · 1 per subnet
              does NOT filter traffic (SG/NACL do)
IGW vs NAT    IGW = in+out, free, attached to VPC · NAT = out only, in PUBLIC subnet + EIP, costs $$
SG vs NACL    SG: instance level, ALLOW only, stateful, all rules evaluated, default deny in
              NACL: subnet level, allow + DENY, stateless, numbered order, default allow all
BASTION ➕    jump box in public subnet · ssh -J · or SSM Session Manager (no port 22)
AWS RESERVES  5 IPs per subnet → /24 = 251 usable
TERRAFORM     aws_vpc · aws_subnet · aws_internet_gateway · aws_route_table + aws_route_table_association
              aws_eip + aws_nat_gateway · aws_security_group · aws_instance · destroy after the lab
```

---

## 14. Likely Exam / Interview Questions

1. **Explain IaaS, PaaS and SaaS with an AWS example of each.** EC2 (you manage the OS and above) / Elastic Beanstalk (you manage only the app) / a SaaS product like Gmail (you just use it).
2. **What is a VPC?** A logically isolated private network in AWS with your own IP range, subnets, routing and firewalls.
3. **Region vs. availability zone? How do VPCs and subnets relate to them?** A region is a geographic area. An AZ is an isolated data centre inside it. A VPC spans a region, and each subnet is in one AZ.
4. **What makes a subnet public?** Its route table has a route `0.0.0.0/0 → Internet Gateway`.
5. **Internet Gateway vs. NAT Gateway?** An IGW allows two-way internet access for public subnets. A NAT Gateway, placed in a public subnet, gives private subnets outbound-only access.
6. **Why put a database in a private subnet?** So it can't be reached directly from the internet. Only the app tier can reach it, through security groups.
7. **What is a route table? Does it block traffic?** Destination → target rules that choose the next hop. No, filtering is done by security groups and NACLs.
8. **Security group vs. NACL?** A security group works at instance level, allow-only and stateful. A NACL works at subnet level, allow and deny, stateless and in numbered order.
9. **Can you write a "deny" rule in a security group?** No. Anything not allowed is denied. Use a NACL for explicit denies.
10. **Why does a NACL need rules for ephemeral ports but a security group doesn't?** NACLs are stateless, so return traffic must be allowed explicitly. Security groups are stateful.
11. **How many usable IPs are in a /24 subnet in AWS?** 251, because AWS reserves 5.
12. **What is a bastion host, and what's a modern alternative?** A hardened jump server in a public subnet for SSH to private instances. The alternative is SSM Session Manager.
13. **EC2 vs. S3 vs. DynamoDB vs. IAM in one line each?** Virtual servers / object storage / managed NoSQL database / access control.
14. **DynamoDB vs. RDS?** NoSQL key-value with a flexible schema and ms latency at scale vs. relational SQL with fixed schemas and JOINs.
15. **Name some S3 storage classes and when you'd use them.** Standard (frequent access), Intelligent-Tiering (unknown access patterns), Standard-IA (rarely accessed), Glacier (archives).
16. **IAM user vs. role? Why use an instance role on EC2?** A user has long-term credentials. A role grants temporary credentials, so there are no access keys stored on the server.
17. **The class Terraform created a public subnet and an IGW, but the instances can't reach the internet. Why?** There's no route table with `0.0.0.0/0 → IGW` associated with the subnet.
18. **Which VPC component costs money even when idle, and how do you avoid surprise bills?** The NAT Gateway. Make it optional, and `terraform destroy` after labs.
