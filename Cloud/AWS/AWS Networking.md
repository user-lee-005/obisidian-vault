# AWS Networking

> **Networking is the backbone of everything in AWS.**
> Every single AWS service — EC2, RDS, Lambda, S3 — communicates over a network.
> Understanding networking is **not optional**. If you skip this, nothing else will make sense.
>
> This note covers how networking works **specifically in AWS**.
> For general cloud networking concepts (IP addresses, CIDR, DNS, etc.), see [[Cloud Networking Basics]].

---

## Table of Contents

- [[#VPC (Virtual Private Cloud)]]
- [[#CIDR Blocks Explained]]
- [[#VPC Hands-On]]
- [[#Subnets]]
- [[#Subnet Hands-On]]
- [[#Internet Gateway (IGW)]]
- [[#NAT Gateway]]
- [[#Route Tables]]
- [[#Security Groups]]
- [[#Network ACLs (NACLs)]]
- [[#Security Groups vs NACLs]]
- [[#Elastic Load Balancing (ELB)]]
- [[#Route 53 (DNS)]]
- [[#VPC Peering]]
- [[#Complete VPC Architecture]]

---

## VPC (Virtual Private Cloud)

### What is a VPC?

A VPC is **your own private, isolated network inside AWS**. Think of it as your own data center in the cloud.

> [!example] Analogy
> Imagine AWS is a massive office building. A VPC is **your own floor** in that building. You control:
> - How many rooms (subnets) you have
> - Who can enter (security groups, NACLs)
> - Which rooms connect to the outside world (internet gateway)
> - The internal phone system (routing)

### Key VPC Facts

- Every AWS account gets a **Default VPC** in each region (pre-configured, internet-accessible)
- You can create **Custom VPCs** with your own IP ranges and configuration
- A VPC lives in **one region** but can span **multiple Availability Zones**
- Resources in a VPC are **isolated** from other VPCs by default
- You choose the **IP address range** (CIDR block) when creating a VPC

### Default VPC vs Custom VPC

| Feature | Default VPC | Custom VPC |
|---|---|---|
| **Created by** | AWS (automatically) | You |
| **CIDR block** | 172.31.0.0/16 | You choose |
| **Subnets** | One public per AZ | You define |
| **Internet access** | Yes (all subnets) | You configure |
| **Use case** | Quick testing | Production workloads |
| **Security** | Less controlled | Full control |

> [!warning] Production Rule
> **Never use the Default VPC for production.** Always create a custom VPC.
> The default VPC has everything open by default — that's insecure.

---

## CIDR Blocks Explained

CIDR (Classless Inter-Domain Routing) defines your IP address range.

### What Does 10.0.0.0/16 Mean?

```
10.0.0.0/16

10.0.0.0  = Starting IP address
/16       = The first 16 bits are fixed (network part)
            The remaining 16 bits are yours (host part)
            2^16 = 65,536 possible IP addresses
```

### Common CIDR Blocks

| CIDR | IP Range | Number of IPs | Use Case |
|---|---|---|---|
| 10.0.0.0/16 | 10.0.0.0 – 10.0.255.255 | 65,536 | VPC (large) |
| 10.0.0.0/20 | 10.0.0.0 – 10.0.15.255 | 4,096 | VPC (medium) |
| 10.0.1.0/24 | 10.0.1.0 – 10.0.1.255 | 256 | Subnet |
| 10.0.1.0/28 | 10.0.1.0 – 10.0.1.15 | 16 | Small subnet |

> [!note] AWS Reserved IPs
> In every subnet, AWS reserves **5 IP addresses**:
> - `.0` — Network address
> - `.1` — VPC router
> - `.2` — DNS server
> - `.3` — Future use
> - `.255` — Broadcast (not supported but reserved)
>
> So a /24 subnet (256 IPs) actually gives you **251 usable IPs**.

### Choosing Your CIDR

For learning, use **10.0.0.0/16**. It gives you 65,536 IPs — more than enough.

For production, plan carefully:
- Don't overlap with your on-premises network
- Don't overlap with other VPCs you might peer with
- Leave room for growth
- Common patterns:
  - Dev: `10.1.0.0/16`
  - Staging: `10.2.0.0/16`
  - Prod: `10.3.0.0/16`

---

## VPC Hands-On

### Create a Custom VPC (CLI)

```bash
# Step 1: Create the VPC
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=MyVPC}]'

# Note the VpcId from the output (e.g., vpc-0abc123def456)
# You'll use this in every subsequent command

# Step 2: Enable DNS hostnames (needed for RDS, ELB, etc.)
aws ec2 modify-vpc-attribute \
  --vpc-id vpc-xxx \
  --enable-dns-hostnames '{"Value":true}'

# Step 3: Enable DNS resolution (usually enabled by default)
aws ec2 modify-vpc-attribute \
  --vpc-id vpc-xxx \
  --enable-dns-support '{"Value":true}'

# Step 4: Verify
aws ec2 describe-vpcs \
  --vpc-ids vpc-xxx \
  --query 'Vpcs[0].{CIDR:CidrBlock,State:State,DNSHostnames:EnableDnsHostnames}'
```

### Create a VPC (Console Steps)

1. Go to **VPC** service → **Your VPCs** → **Create VPC**
2. Select **VPC only** (not "VPC and more" — we want to learn each piece)
3. Name: `MyVPC`
4. IPv4 CIDR: `10.0.0.0/16`
5. Click **Create VPC**
6. Select your new VPC → **Actions** → **Edit VPC settings** → Enable **DNS hostnames**

> [!tip] The "VPC and more" wizard
> AWS has a wizard that creates VPC + subnets + IGW + NAT + route tables all at once.
> It's great for production, but learn each component separately first.

---

## Subnets

### What is a Subnet?

A subnet is a **sub-division of your VPC's IP range**. It lives in **one specific Availability Zone**.

Think of your VPC as a building. Subnets are **floors** — each floor has a specific purpose:

```
VPC: 10.0.0.0/16 (the building)
├── Public Subnet 1a:  10.0.1.0/24  (AZ: ap-south-1a) — Front desk, open to visitors
├── Public Subnet 1b:  10.0.3.0/24  (AZ: ap-south-1b) — Front desk, backup
├── Private Subnet 1a: 10.0.2.0/24  (AZ: ap-south-1a) — Back office, staff only
└── Private Subnet 1b: 10.0.4.0/24  (AZ: ap-south-1b) — Back office, backup
```

### Public vs Private Subnets

| Feature | Public Subnet | Private Subnet |
|---|---|---|
| **Route to IGW** | Yes (0.0.0.0/0 → IGW) | No |
| **Public IP** | Instances can get public IPs | No public IPs |
| **Internet access** | Full (inbound + outbound) | Outbound only (via NAT) |
| **What goes here** | Load balancers, bastion hosts | App servers, databases |

> [!important] What Makes a Subnet "Public"?
> A subnet is NOT automatically public. It becomes public when:
> 1. Its route table has a route to an Internet Gateway (0.0.0.0/0 → IGW)
> 2. Instances in it have a public IP address
>
> Without BOTH, it's effectively private.

### Best Practice: Multi-AZ

Always create subnets in **at least 2 Availability Zones**:
- If one AZ goes down, your app keeps running in the other AZ
- Load balancers REQUIRE subnets in multiple AZs
- RDS Multi-AZ needs subnets in different AZs

---

## Subnet Hands-On

### Create Subnets (CLI)

```bash
# Public subnet in AZ 1a
aws ec2 create-subnet \
  --vpc-id vpc-xxx \
  --cidr-block 10.0.1.0/24 \
  --availability-zone ap-south-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Public-1a}]'

# Public subnet in AZ 1b
aws ec2 create-subnet \
  --vpc-id vpc-xxx \
  --cidr-block 10.0.3.0/24 \
  --availability-zone ap-south-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Public-1b}]'

# Private subnet in AZ 1a
aws ec2 create-subnet \
  --vpc-id vpc-xxx \
  --cidr-block 10.0.2.0/24 \
  --availability-zone ap-south-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Private-1a}]'

# Private subnet in AZ 1b
aws ec2 create-subnet \
  --vpc-id vpc-xxx \
  --cidr-block 10.0.4.0/24 \
  --availability-zone ap-south-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Private-1b}]'

# Enable auto-assign public IP for public subnets
aws ec2 modify-subnet-attribute \
  --subnet-id subnet-public-1a-xxx \
  --map-public-ip-on-launch

aws ec2 modify-subnet-attribute \
  --subnet-id subnet-public-1b-xxx \
  --map-public-ip-on-launch

# Verify all subnets
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=vpc-xxx" \
  --query 'Subnets[*].{Name:Tags[?Key==`Name`].Value|[0],AZ:AvailabilityZone,CIDR:CidrBlock}' \
  --output table
```

---

## Internet Gateway (IGW)

### What is an Internet Gateway?

An IGW is the **door between your VPC and the public internet**. Without it, nothing in your VPC can reach the internet (and vice versa).

```
Internet ←→ Internet Gateway ←→ Public Subnet ←→ Your EC2 Instance
```

Key facts:
- **One IGW per VPC** (one-to-one relationship)
- It's **horizontally scaled, redundant, and highly available** — AWS manages it
- It's **free** — no charge for the IGW itself
- You still need a **route table entry** pointing to the IGW

### IGW Hands-On

```bash
# Step 1: Create the Internet Gateway
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=MyIGW}]'

# Note the InternetGatewayId (e.g., igw-0abc123)

# Step 2: Attach it to your VPC
aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-xxx \
  --vpc-id vpc-xxx

# Step 3: Verify
aws ec2 describe-internet-gateways \
  --internet-gateway-ids igw-xxx \
  --query 'InternetGateways[0].Attachments'
```

> [!note] Just attaching an IGW isn't enough
> You also need to:
> 1. Create a route table with a route to the IGW
> 2. Associate that route table with your public subnets
> 3. Ensure instances have public IPs
> 4. Ensure security groups allow the traffic

---

## NAT Gateway

### What is a NAT Gateway?

NAT (Network Address Translation) Gateway lets resources in **private subnets** access the internet **without being accessible FROM the internet**.

```
Private Subnet Instance
    → NAT Gateway (in public subnet)
        → Internet Gateway
            → Internet (e.g., download updates)

Internet → Cannot reach private subnet instances (one-way only)
```

### Why Do You Need NAT?

Your private subnet EC2 instances need internet access for:
- Downloading software updates (`yum update`, `apt upgrade`)
- Pulling Docker images
- Calling external APIs
- Sending logs to external services

But you **don't** want them directly exposed to the internet.

### NAT Gateway vs NAT Instance

| Feature | NAT Gateway (Recommended) | NAT Instance (Old way) |
|---|---|---|
| **Managed by** | AWS | You |
| **Availability** | Highly available in AZ | You set up failover |
| **Bandwidth** | Up to 100 Gbps | Depends on instance type |
| **Maintenance** | None | You patch and update |
| **Cost** | ~$0.045/hr + data | Instance cost |
| **Security Groups** | Not applicable | You manage SGs |

> [!tip] Always use **NAT Gateway**, never NAT Instance. NAT Instance is a legacy approach.

### NAT Gateway Hands-On

```bash
# Step 1: Allocate an Elastic IP (NAT Gateway needs a static IP)
aws ec2 allocate-address --domain vpc
# Note the AllocationId (e.g., eipalloc-0abc123)

# Step 2: Create NAT Gateway in a PUBLIC subnet
aws ec2 create-nat-gateway \
  --subnet-id subnet-public-1a-xxx \
  --allocation-id eipalloc-xxx \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=MyNATGW}]'

# Step 3: Wait for it to be available (takes 1-2 minutes)
aws ec2 wait nat-gateway-available --nat-gateway-ids nat-xxx

# Step 4: Add a route in the PRIVATE subnet's route table
# (Traffic going to internet → send to NAT Gateway)
aws ec2 create-route \
  --route-table-id rtb-private-xxx \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-xxx
```

> [!warning] NAT Gateway Costs
> NAT Gateway charges per hour (~$0.045/hr = ~$32/month) PLUS data processing charges.
> **Delete it when not in use** during learning! See [[AWS Billing]].

---

## Route Tables

### What is a Route Table?

A route table is a **set of rules (routes)** that determine where network traffic goes.

Every subnet **must** be associated with a route table. If you don't explicitly associate one, it uses the **main route table** of the VPC.

### How Routes Work

```
Route Table for Public Subnet:
+-------------------+----------+
| Destination       | Target   |
+-------------------+----------+
| 10.0.0.0/16      | local    |   ← Traffic within VPC stays within VPC
| 0.0.0.0/0        | igw-xxx  |   ← Everything else → Internet Gateway
+-------------------+----------+

Route Table for Private Subnet:
+-------------------+----------+
| Destination       | Target   |
+-------------------+----------+
| 10.0.0.0/16      | local    |   ← Traffic within VPC stays within VPC
| 0.0.0.0/0        | nat-xxx  |   ← Everything else → NAT Gateway
+-------------------+----------+
```

The `local` route is automatic and can't be deleted — it allows all subnets in the VPC to communicate with each other.

### Route Table Hands-On

```bash
# Create route table for public subnets
aws ec2 create-route-table \
  --vpc-id vpc-xxx \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Public-RT}]'

# Add route to Internet Gateway (makes it a "public" route table)
aws ec2 create-route \
  --route-table-id rtb-public-xxx \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-xxx

# Associate with public subnets
aws ec2 associate-route-table \
  --route-table-id rtb-public-xxx \
  --subnet-id subnet-public-1a-xxx

aws ec2 associate-route-table \
  --route-table-id rtb-public-xxx \
  --subnet-id subnet-public-1b-xxx

# Create route table for private subnets
aws ec2 create-route-table \
  --vpc-id vpc-xxx \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Private-RT}]'

# Add route to NAT Gateway
aws ec2 create-route \
  --route-table-id rtb-private-xxx \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-xxx

# Associate with private subnets
aws ec2 associate-route-table \
  --route-table-id rtb-private-xxx \
  --subnet-id subnet-private-1a-xxx

aws ec2 associate-route-table \
  --route-table-id rtb-private-xxx \
  --subnet-id subnet-private-1b-xxx

# Verify routes
aws ec2 describe-route-tables \
  --route-table-ids rtb-public-xxx \
  --query 'RouteTables[0].Routes'
```

---

## Security Groups

### What is a Security Group?

A security group is a **virtual firewall for your EC2 instances** (and other resources). It controls **inbound** (incoming) and **outbound** (outgoing) traffic.

### Key Characteristics

- **Stateful** — If you allow inbound traffic, the response is automatically allowed out (and vice versa)
- **Allow rules only** — You can only specify what to ALLOW, not what to DENY
- **Default behavior** — All inbound traffic is DENIED, all outbound traffic is ALLOWED
- **Instance-level** — Attached to individual instances (technically to network interfaces)
- **Can reference other SGs** — Instead of IP ranges, you can say "allow traffic from SG-xyz"

### Security Group Rules Example

```
Web Server Security Group (web-sg):
┌──────────┬──────────┬──────┬────────────────┐
│ Type     │ Protocol │ Port │ Source         │
├──────────┼──────────┼──────┼────────────────┤
│ Inbound  │ TCP      │ 80   │ 0.0.0.0/0     │  ← HTTP from anywhere
│ Inbound  │ TCP      │ 443  │ 0.0.0.0/0     │  ← HTTPS from anywhere
│ Inbound  │ TCP      │ 22   │ 203.0.113.5/32│  ← SSH from your IP only
│ Outbound │ All      │ All  │ 0.0.0.0/0     │  ← Allow all outbound
└──────────┴──────────┴──────┴────────────────┘

Database Security Group (db-sg):
┌──────────┬──────────┬──────┬────────────────┐
│ Type     │ Protocol │ Port │ Source         │
├──────────┼──────────┼──────┼────────────────┤
│ Inbound  │ TCP      │ 3306 │ sg-web-xxx    │  ← MySQL from web SG only
│ Outbound │ All      │ All  │ 0.0.0.0/0     │  ← Allow all outbound
└──────────┴──────────┴──────┴────────────────┘
```

> [!important] Security Group Best Practice
> **Never allow 0.0.0.0/0 on SSH (port 22)**. Always restrict to your IP address.
> Leaving SSH open to the world is the #1 security mistake beginners make.

### Security Group Hands-On

```bash
# Create a security group for web servers
aws ec2 create-security-group \
  --group-name web-sg \
  --description "Security group for web servers" \
  --vpc-id vpc-xxx

# Allow SSH from your IP only
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxx \
  --protocol tcp \
  --port 22 \
  --cidr 203.0.113.5/32

# Allow HTTP from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxx \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Allow HTTPS from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxx \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Create a security group for databases
aws ec2 create-security-group \
  --group-name db-sg \
  --description "Security group for databases" \
  --vpc-id vpc-xxx

# Allow MySQL (3306) ONLY from the web security group
aws ec2 authorize-security-group-ingress \
  --group-id sg-db-xxx \
  --protocol tcp \
  --port 3306 \
  --source-group sg-web-xxx

# List rules for a security group
aws ec2 describe-security-group-rules \
  --filters "Name=group-id,Values=sg-xxx" \
  --query 'SecurityGroupRules[*].{Direction:IsEgress,Protocol:IpProtocol,Port:FromPort,Source:CidrIpv4||ReferencedGroupInfo.GroupId}' \
  --output table
```

---

## Network ACLs (NACLs)

### What is a NACL?

A Network ACL is a **firewall at the subnet level**. It's an extra layer of defense.

### Key Characteristics

- **Stateless** — Inbound and outbound rules are evaluated independently (you must explicitly allow both directions)
- **Allow AND Deny rules** — You can explicitly block specific IPs
- **Subnet-level** — Applies to ALL resources in the subnet
- **Rules are numbered** — Evaluated in order, lowest number first
- **Default NACL** — Allows all inbound and outbound traffic

### When to Use NACLs

- Block a specific IP address (e.g., an attacker)
- Add a subnet-level firewall in addition to security groups
- Compliance requirements that mandate network-level controls

---

## Security Groups vs NACLs

| Feature | Security Groups | Network ACLs |
|---|---|---|
| **Level** | Instance (ENI) | Subnet |
| **Stateful/Stateless** | Stateful | Stateless |
| **Rule Type** | Allow only | Allow AND Deny |
| **Rule Evaluation** | All rules evaluated | Rules evaluated in number order |
| **Default** | Deny all inbound, allow all outbound | Allow all (default NACL) |
| **Applies to** | Only instances assigned to the SG | All instances in the subnet |
| **Association** | Can belong to multiple SGs | One NACL per subnet |
| **Return traffic** | Automatically allowed | Must be explicitly allowed |

> [!example] Analogy
> - **Security Group** = Lock on your apartment door (per-apartment security)
> - **NACL** = Security guard at the building entrance (building-wide security)
>
> Most of the time, **Security Groups are sufficient**. Use NACLs for extra defense.

---

## Elastic Load Balancing (ELB)

### What is a Load Balancer?

A load balancer **distributes incoming traffic** across multiple targets (EC2 instances, containers, etc.) so no single target gets overwhelmed.

```
         Users
           │
    ┌──────▼──────┐
    │ Load Balancer│
    └──────┬──────┘
       ┌───┼───┐
       ▼   ▼   ▼
     EC2  EC2  EC2
      1    2    3
```

### Why Use a Load Balancer?

1. **Distribute traffic** — No single server gets overloaded
2. **High availability** — If one server dies, traffic goes to healthy servers
3. **Health checks** — Automatically detect and remove unhealthy targets
4. **SSL termination** — Handle HTTPS at the load balancer, not each server
5. **Single endpoint** — Users connect to one DNS name, not individual servers

### Types of Load Balancers

| Type | Layer | Protocol | Best For |
|---|---|---|---|
| **ALB** (Application) | Layer 7 | HTTP, HTTPS, gRPC | Web applications, microservices |
| **NLB** (Network) | Layer 4 | TCP, UDP, TLS | Gaming, IoT, ultra-low latency |
| **GLB** (Gateway) | Layer 3 | IP | Firewalls, IDS/IPS appliances |

### ALB (Application Load Balancer) — Most Common

The ALB operates at **Layer 7 (HTTP)**, which means it understands:
- URLs and paths → Route `/api/*` to one set of servers, `/web/*` to another
- HTTP headers → Route based on `Host:` header
- Query strings → Route based on URL parameters

```
ALB Routing Example:
Users → ALB
         ├── /api/*    → Target Group: API Servers (port 8080)
         ├── /web/*    → Target Group: Web Servers (port 80)
         └── /images/* → Target Group: CDN/S3 (redirect)
```

### NLB (Network Load Balancer)

The NLB operates at **Layer 4 (TCP/UDP)**:
- **Ultra-low latency** (millions of requests per second)
- **Static IP** support (one per AZ)
- Preserves the source IP address
- Best for non-HTTP protocols (gaming, IoT, real-time communications)

### Target Groups

A target group is a **collection of targets** (instances, IPs, or Lambda functions) that receive traffic from the load balancer.

```bash
# Create a target group
aws elbv2 create-target-group \
  --name my-web-targets \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-xxx \
  --health-check-protocol HTTP \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 3 \
  --unhealthy-threshold-count 2

# Register EC2 instances
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/my-web-targets/xxx \
  --targets Id=i-111 Id=i-222 Id=i-333
```

### Health Checks

The load balancer periodically checks if targets are healthy:

| Setting | Description | Example |
|---|---|---|
| **Protocol** | HTTP, HTTPS, TCP | HTTP |
| **Path** | URL to check | `/health` |
| **Interval** | How often to check | 30 seconds |
| **Healthy threshold** | Consecutive successes to be healthy | 3 |
| **Unhealthy threshold** | Consecutive failures to be unhealthy | 2 |
| **Timeout** | How long to wait for response | 5 seconds |

### ALB Hands-On

```bash
# Step 1: Create the ALB (must be in at least 2 AZ subnets)
aws elbv2 create-load-balancer \
  --name my-alb \
  --type application \
  --subnets subnet-public-1a-xxx subnet-public-1b-xxx \
  --security-groups sg-alb-xxx \
  --scheme internet-facing

# Step 2: Create target group (see above)

# Step 3: Create a listener (defines what port ALB listens on)
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/app/my-alb/xxx \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...:targetgroup/my-web-targets/xxx

# Step 4: Verify health of targets
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/my-web-targets/xxx

# Step 5: Get the ALB DNS name (this is what users connect to)
aws elbv2 describe-load-balancers \
  --names my-alb \
  --query 'LoadBalancers[0].DNSName' \
  --output text
```

---

## Route 53 (DNS)

### What is Route 53?

Route 53 is AWS's **DNS (Domain Name System)** service. It translates human-readable domain names into IP addresses.

```
User types: www.myapp.com
    → Route 53 resolves it to: 54.123.45.67 (your ALB or EC2)
        → Browser connects to that IP
```

### Why "Route 53"?

DNS uses **port 53**. And "Route" refers to routing traffic. Hence, Route 53.

### Key Concepts

| Concept | Description |
|---|---|
| **Hosted Zone** | Container for DNS records for one domain (e.g., myapp.com) |
| **Record** | A single DNS entry (e.g., www → 54.123.45.67) |
| **TTL** | Time To Live — how long to cache the DNS response |

### DNS Record Types

| Type | Purpose | Example |
|---|---|---|
| **A** | Maps domain to IPv4 address | www.myapp.com → 54.123.45.67 |
| **AAAA** | Maps domain to IPv6 address | www.myapp.com → 2001:db8::1 |
| **CNAME** | Maps domain to another domain | api.myapp.com → my-alb.amazonaws.com |
| **MX** | Mail server records | myapp.com → mail.myapp.com |
| **TXT** | Text records (verification, SPF) | myapp.com → "v=spf1 include:..." |
| **Alias** | AWS-specific, maps to AWS resources | myapp.com → ALB/S3/CloudFront (free!) |

> [!tip] Use Alias Records for AWS Resources
> Alias records are free (no charge for queries) and work at the zone apex (e.g., `myapp.com` without `www`).
> CNAME records cannot be used at the zone apex and cost money per query.

### Routing Policies

| Policy | How It Works | Use Case |
|---|---|---|
| **Simple** | One record, one or more IPs | Basic routing |
| **Weighted** | Split traffic by percentage | A/B testing (90% v1, 10% v2) |
| **Latency** | Route to lowest-latency region | Multi-region apps |
| **Failover** | Primary/secondary with health checks | Disaster recovery |
| **Geolocation** | Route based on user's location | Show region-specific content |
| **Multi-value** | Return multiple healthy IPs | Simple load balancing |

---

## VPC Peering

### What is VPC Peering?

VPC Peering creates a **private network connection between two VPCs**. Traffic stays on the AWS backbone (never touches the internet).

```
VPC A (10.0.0.0/16) ←──VPC Peering──→ VPC B (172.16.0.0/16)
    EC2 in A can talk to RDS in B privately
```

### Key Rules

- **No transitive peering** — If A peers with B, and B peers with C, A CANNOT talk to C through B
- **No overlapping CIDR** — The VPCs must have different IP ranges
- **Can cross accounts and regions** — Peer VPCs in different AWS accounts or regions
- **Both sides must accept** — Requester creates, accepter approves

### VPC Peering Hands-On

```bash
# Step 1: Create peering connection
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-A-xxx \
  --peer-vpc-id vpc-B-xxx \
  --peer-region ap-south-1

# Step 2: Accept the peering (from the other account/VPC)
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id pcx-xxx

# Step 3: Add routes in BOTH VPCs
# In VPC A's route table: traffic to VPC B → peering connection
aws ec2 create-route \
  --route-table-id rtb-A-xxx \
  --destination-cidr-block 172.16.0.0/16 \
  --vpc-peering-connection-id pcx-xxx

# In VPC B's route table: traffic to VPC A → peering connection
aws ec2 create-route \
  --route-table-id rtb-B-xxx \
  --destination-cidr-block 10.0.0.0/16 \
  --vpc-peering-connection-id pcx-xxx

# Step 4: Update security groups in BOTH VPCs to allow traffic
```

> [!note] For Many VPCs
> If you need to connect many VPCs, use **AWS Transit Gateway** instead of peering.
> Transit Gateway acts as a central hub — all VPCs connect to it (like a star topology).

---

## Complete VPC Architecture

Here's a **typical production VPC setup** that ties everything together:

```
┌─────────────────────── VPC: 10.0.0.0/16 ───────────────────────┐
│                                                                  │
│   ┌──── AZ: ap-south-1a ────┐    ┌──── AZ: ap-south-1b ────┐  │
│   │                          │    │                          │  │
│   │  ┌─ Public 10.0.1.0/24─┐│    │┌─ Public 10.0.3.0/24──┐ │  │
│   │  │  NAT Gateway         ││    ││  NAT Gateway          │ │  │
│   │  │  ALB node            ││    ││  ALB node             │ │  │
│   │  │  Bastion Host        ││    ││                       │ │  │
│   │  └──────────────────────┘│    │└────────────────────────┘ │  │
│   │                          │    │                          │  │
│   │  ┌─ Private 10.0.2.0/24┐│    │┌─ Private 10.0.4.0/24─┐ │  │
│   │  │  EC2 App Server 1   ││    ││  EC2 App Server 2     │ │  │
│   │  │  ECS/Fargate Tasks  ││    ││  ECS/Fargate Tasks    │ │  │
│   │  └──────────────────────┘│    │└────────────────────────┘ │  │
│   │                          │    │                          │  │
│   │  ┌─ Private 10.0.5.0/24┐│    │┌─ Private 10.0.6.0/24─┐ │  │
│   │  │  RDS Primary         ││    ││  RDS Standby (Multi-AZ)│ │  │
│   │  │  ElastiCache Primary ││    ││  ElastiCache Replica  │ │  │
│   │  └──────────────────────┘│    │└────────────────────────┘ │  │
│   │                          │    │                          │  │
│   └──────────────────────────┘    └──────────────────────────┘  │
│                                                                  │
│   Internet Gateway (igw-xxx)                                     │
└──────────────────────────────────────────────────────────────────┘
                    │
              ┌─────▼─────┐
              │  Internet  │
              └───────────┘

Traffic Flow:
1. User → Internet → IGW → ALB (public subnet)
2. ALB → EC2/ECS (private subnet) — health checked
3. EC2 → RDS/ElastiCache (private data subnet) — SG restricted
4. EC2 → NAT Gateway → IGW → Internet (for updates)
5. RDS → Multi-AZ standby (synchronous replication)
```

### What Each Layer Does

| Layer | Subnet Type | Components | Purpose |
|---|---|---|---|
| **DMZ** | Public | ALB, NAT GW, Bastion | Internet-facing entry point |
| **Application** | Private | EC2, ECS, Lambda | Business logic |
| **Data** | Private | RDS, ElastiCache, DynamoDB | Data storage |

### Building This Architecture (Summary of CLI Steps)

```bash
# 1. Create VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16

# 2. Create 6 subnets (2 public, 2 app-private, 2 data-private)
# 3. Create and attach Internet Gateway
# 4. Create NAT Gateway in each public subnet
# 5. Create route tables (public → IGW, private → NAT)
# 6. Associate route tables with subnets
# 7. Create security groups (ALB-sg, app-sg, db-sg)
# 8. Deploy ALB in public subnets
# 9. Deploy EC2/ECS in private app subnets
# 10. Deploy RDS in private data subnets

# Or use the "VPC and more" wizard in the console for a shortcut!
```

> [!tip] Infrastructure as Code
> In real production, you wouldn't run these CLI commands manually.
> You'd use **AWS CloudFormation** or **Terraform** to define all of this in code.
> That way, you can version control your infrastructure and recreate it reliably.

---

## Summary

| Component | What It Does | Key Point |
|---|---|---|
| **VPC** | Your private network | Always create custom VPCs for production |
| **Subnets** | Divide VPC into zones | Public (internet) and Private (internal) |
| **IGW** | Internet access for VPC | Attach to VPC, route from public subnets |
| **NAT Gateway** | Internet for private subnets | One-way outbound only |
| **Route Tables** | Traffic direction rules | Public → IGW, Private → NAT |
| **Security Groups** | Instance firewall (stateful) | Allow rules only, default deny inbound |
| **NACLs** | Subnet firewall (stateless) | Allow + Deny rules, numbered order |
| **ALB** | HTTP/HTTPS load balancer | Path-based routing, health checks |
| **NLB** | TCP/UDP load balancer | Ultra-low latency, static IPs |
| **Route 53** | DNS service | Domain registration, routing policies |
| **VPC Peering** | Connect two VPCs | No transitive peering |

---

## Cross-References

- [[Cloud Networking Basics]] — General networking concepts (IP, CIDR, DNS, TCP/IP)
- [[AWS Getting Started]] — Setting up your AWS account
- [[AWS Compute]] — EC2 instances that live inside your VPC
- [[AWS Databases]] — RDS and ElastiCache that run in private subnets
- [[Azure Networking]] — How Azure handles networking (for comparison)
- [[AWS Billing]] — NAT Gateway and data transfer costs
