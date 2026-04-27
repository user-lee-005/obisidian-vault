# Cloud Networking Basics

> [!info] Prerequisites
> Read [[Cloud Fundamentals]] first to understand the basics of cloud computing before diving into networking.

---

## Why Cloud Networking Matters

Every cloud resource — every virtual machine, database, container, and function — **communicates over a network**. Understanding cloud networking is essential because:

- It controls **who can access** your resources (security)
- It determines **how fast** your resources communicate (performance)
- It defines **how your architecture** is structured (design)
- Misconfigured networking is a **top cause of cloud security breaches**

> [!warning] Real-World Impact
> In 2019, Capital One suffered a massive data breach because of a misconfigured firewall rule in their AWS environment. Understanding networking could have prevented it.

If you don't understand networking, you can't build secure, reliable, or performant cloud architectures.

---

## Virtual Private Cloud (VPC / VNet)

### What Is It?

A **Virtual Private Cloud (VPC)** is your own **isolated, private network** inside the cloud provider's infrastructure. Think of it as your own private section of the internet that only you control.

- **AWS** calls it a **VPC** (Virtual Private Cloud)
- **Azure** calls it a **VNet** (Virtual Network)
- **GCP** calls it a **VPC Network**

### Why Do You Need One?

Without a VPC, your resources would sit on a shared, public network — visible and accessible to everyone. A VPC gives you:

- **Isolation** — Your resources are separated from other customers.
- **Control** — You define IP ranges, subnets, route tables, and gateways.
- **Security** — You control who can talk to whom using firewalls.

### The House Analogy

Think of a VPC as a **gated community**:
- The community has walls (isolation from the internet)
- Inside, there are streets and house numbers (subnets and IP addresses)
- There's a main gate to the outside world (Internet Gateway)
- There are security guards checking who enters and leaves (firewalls)

### CIDR Notation — Defining Your Network Size

When you create a VPC, you must specify an **IP address range** using **CIDR notation** (Classless Inter-Domain Routing).

**Don't panic** — it's simpler than it sounds.

**Format:** `10.0.0.0/16`

- `10.0.0.0` → The starting IP address
- `/16` → How many IP addresses are available

**The `/number` (prefix length) determines the size:**

| CIDR | Available IPs | Use Case |
|---|---|---|
| `/16` | 65,536 IPs | Large VPC (recommended for production) |
| `/20` | 4,096 IPs | Medium subnet |
| `/24` | 256 IPs | Small subnet (common for individual subnets) |
| `/28` | 16 IPs | Very small subnet |
| `/32` | 1 IP | Single host |

> [!tip] Rule of Thumb
> The **smaller the number after `/`**, the **larger the network**. `/16` is huge, `/32` is a single IP.

**Common private IP ranges (RFC 1918):**
- `10.0.0.0/8` → 10.x.x.x (most common in cloud)
- `172.16.0.0/12` → 172.16.x.x to 172.31.x.x
- `192.168.0.0/16` → 192.168.x.x (common in home networks)

**Example:** Creating a VPC with `10.0.0.0/16` gives you all IPs from `10.0.0.0` to `10.0.255.255` — that's 65,536 addresses to assign to your resources.

---

## Subnets

### What Is a Subnet?

A **subnet** is a **smaller network inside your VPC**. You divide your VPC into subnets to organise and isolate resources.

Think of it this way:
- **VPC** = Your gated community
- **Subnet** = A neighbourhood within the community

### Public vs Private Subnets

| Aspect | Public Subnet | Private Subnet |
|---|---|---|
| **Internet access** | Direct (via Internet Gateway) | No direct access |
| **What goes here** | Web servers, load balancers, bastion hosts | Databases, application servers, internal APIs |
| **Has public IP?** | Yes (or Elastic IP) | No |
| **Accessible from internet?** | Yes | No (by design) |
| **Can reach internet?** | Yes | Only via NAT Gateway |

### Why Separate Them?

**Security.** You don't want your database directly accessible from the internet. By putting it in a private subnet, the only way to reach it is through your application servers in the public subnet.

### Typical Subnet Layout

```
VPC: 10.0.0.0/16
│
├── Public Subnet A:  10.0.1.0/24  (AZ-1)  → Web servers, Load Balancers
├── Public Subnet B:  10.0.2.0/24  (AZ-2)  → Web servers, Load Balancers
│
├── Private Subnet A: 10.0.10.0/24 (AZ-1)  → App servers, APIs
├── Private Subnet B: 10.0.11.0/24 (AZ-2)  → App servers, APIs
│
├── Data Subnet A:    10.0.20.0/24 (AZ-1)  → Databases
└── Data Subnet B:    10.0.21.0/24 (AZ-2)  → Databases
```

> [!tip] Best Practice
> Always create subnets across **multiple Availability Zones** for high availability. If AZ-1 goes down, your resources in AZ-2 keep running.

---

## IP Addressing

### Public vs Private IPs

| Type | Description | Example | Visible on Internet? |
|---|---|---|---|
| **Private IP** | Internal address within your VPC | `10.0.1.45` | No |
| **Public IP** | Address accessible from the internet | `54.23.100.78` | Yes |

- Every resource in a VPC gets a **private IP** automatically.
- Resources in a **public subnet** can also get a **public IP**.
- Resources in a **private subnet** do NOT get a public IP.

### Elastic IP / Static IP

By default, public IPs **change** every time you stop and start a VM. An **Elastic IP** (AWS) or **Static IP** (Azure) is a **fixed public IP** that doesn't change.

**When to use:**
- When you need a permanent IP for DNS records
- When external systems need to whitelist your IP

> [!warning] Cost Note
> Elastic IPs are **free when attached** to a running instance, but you get **charged when they're NOT in use**. Always release unused Elastic IPs.

### IPv4 vs IPv6

| Feature | IPv4 | IPv6 |
|---|---|---|
| **Format** | `192.168.1.1` (4 groups of numbers) | `2001:0db8:85a3::8a2e:0370:7334` (8 groups of hex) |
| **Total addresses** | ~4.3 billion | ~340 undecillion (practically unlimited) |
| **Status** | Running out of addresses | The future, slowly being adopted |
| **Cloud support** | Fully supported everywhere | Supported but optional |

> [!note] For Beginners
> Stick with **IPv4** for now. IPv6 is important but not essential for learning cloud basics.

---

## Gateways

Gateways are the **doors** that connect your VPC to the outside world (or other networks).

### Internet Gateway (IGW)

**What it does:** Connects your VPC to the **public internet**. Without it, nothing in your VPC can reach (or be reached from) the internet.

**Key points:**
- One Internet Gateway per VPC
- Must be attached to the VPC
- Route tables must have a route pointing to it
- Only resources in **public subnets** with public IPs use it

```
Internet  ←→  Internet Gateway  ←→  Public Subnet (Web Servers)
```

### NAT Gateway (Network Address Translation)

**What it does:** Allows resources in **private subnets** to access the internet (e.g., to download software updates) **without exposing them** to incoming internet traffic.

**Key points:**
- Lives in a **public subnet**
- Private subnet route table points to NAT Gateway for outbound traffic
- Traffic is **one-way** — private resources CAN reach the internet, but the internet CANNOT reach them
- Costs money — charged per hour and per GB of data processed

```
Private Subnet (App Server)
   → NAT Gateway (in Public Subnet)
      → Internet Gateway
         → Internet
```

> [!example] Real-World Example
> Your database server is in a private subnet. It needs to download security patches from the internet. It can do this through the NAT Gateway without being directly exposed to the internet.

### VPN Gateway

**What it does:** Creates an **encrypted tunnel** between your cloud VPC and your on-premises data centre (or another network).

**When to use:**
- You have servers on-premises that need to talk to cloud resources
- You need a secure connection without going over the public internet
- Part of a **hybrid cloud** setup (see [[Cloud Fundamentals]])

```
On-Premises Data Centre  ←🔒→  VPN Gateway  ←→  VPC
```

**Cloud services:**
- AWS: **Site-to-Site VPN**, **Client VPN**
- Azure: **VPN Gateway**

---

## Route Tables

### What Is a Route Table?

A **route table** is a set of rules (called **routes**) that determine **where network traffic goes**. Every subnet in your VPC is associated with a route table.

### How It Works

Think of it as a **GPS for your network traffic**:

| Destination | Target | Meaning |
|---|---|---|
| `10.0.0.0/16` | `local` | Traffic within the VPC stays inside |
| `0.0.0.0/0` | `igw-xxxxx` | All other traffic goes to the Internet Gateway |

- `0.0.0.0/0` means **"everything else"** (the default route, like a catch-all)
- `local` means **"stay within the VPC"**

### Public Subnet Route Table

```
Destination      Target
10.0.0.0/16      local          ← VPC internal traffic
0.0.0.0/0        igw-abc123     ← Internet-bound traffic → Internet Gateway
```

### Private Subnet Route Table

```
Destination      Target
10.0.0.0/16      local          ← VPC internal traffic
0.0.0.0/0        nat-xyz789     ← Internet-bound traffic → NAT Gateway (outbound only)
```

> [!important] Key Difference
> The **only difference** between a "public" and "private" subnet is what the **route table** points to for `0.0.0.0/0`:
> - Public subnet → Internet Gateway (direct internet access)
> - Private subnet → NAT Gateway (outbound-only internet access)

---

## Firewalls & Security

Cloud networking has **two layers of firewalls** that work together:

### Security Groups (Stateful Firewall)

**What it is:** A virtual firewall attached to **individual resources** (e.g., a specific VM or database). Controls inbound and outbound traffic.

**Key characteristics:**
- **Stateful** — If you allow traffic IN, the response OUT is automatically allowed (and vice versa).
- Operates at the **instance level** (attached to each resource).
- **Allow rules only** — You specify what to ALLOW. Everything else is denied by default.
- Changes take effect **immediately**.

**Example Security Group for a Web Server:**

| Direction | Protocol | Port | Source | Description |
|---|---|---|---|---|
| Inbound | TCP | 80 | `0.0.0.0/0` | Allow HTTP from anywhere |
| Inbound | TCP | 443 | `0.0.0.0/0` | Allow HTTPS from anywhere |
| Inbound | TCP | 22 | `10.0.0.0/16` | Allow SSH from within VPC only |
| Outbound | All | All | `0.0.0.0/0` | Allow all outbound traffic |

### Network ACLs (Stateless Firewall)

**What it is:** A firewall attached to a **subnet**. Controls traffic entering and leaving the entire subnet.

**Key characteristics:**
- **Stateless** — Inbound and outbound rules are evaluated **independently**. You must explicitly allow BOTH directions.
- Operates at the **subnet level**.
- Has both **Allow and Deny** rules.
- Rules are evaluated in **number order** (lowest number first).

**Example Network ACL:**

| Rule # | Direction | Protocol | Port | Source/Dest | Action |
|---|---|---|---|---|---|
| 100 | Inbound | TCP | 80 | `0.0.0.0/0` | ALLOW |
| 110 | Inbound | TCP | 443 | `0.0.0.0/0` | ALLOW |
| 200 | Inbound | TCP | 22 | `10.0.0.0/16` | ALLOW |
| * | Inbound | All | All | `0.0.0.0/0` | DENY |
| 100 | Outbound | TCP | 80 | `0.0.0.0/0` | ALLOW |
| 110 | Outbound | TCP | 443 | `0.0.0.0/0` | ALLOW |
| * | Outbound | All | All | `0.0.0.0/0` | DENY |

### Security Groups vs Network ACLs — Comparison

| Feature | Security Groups | Network ACLs |
|---|---|---|
| **Level** | Instance (resource) | Subnet |
| **Stateful/Stateless** | Stateful | Stateless |
| **Rules** | Allow only | Allow AND Deny |
| **Default behaviour** | Deny all inbound, allow all outbound | Allow all (default ACL) |
| **Rule evaluation** | All rules evaluated together | Rules evaluated in number order |
| **Use case** | Fine-grained per-resource control | Broad subnet-level control |

> [!tip] Best Practice
> Use **Security Groups** as your primary firewall (they're easier to manage because they're stateful). Use **Network ACLs** as an additional layer of defence for subnet-level blocking.

---

## DNS (Domain Name System)

### What Is DNS?

DNS is the **phone book of the internet**. It translates human-readable domain names (like `www.example.com`) into IP addresses (like `93.184.216.34`) that computers use.

### How DNS Works (Simplified)

```
You type: www.example.com
   ↓
Your browser asks a DNS Resolver: "What's the IP for www.example.com?"
   ↓
DNS Resolver checks its cache, or asks Root Servers → TLD Servers → Authoritative Servers
   ↓
Answer: 93.184.216.34
   ↓
Your browser connects to 93.184.216.34
```

### DNS Record Types

| Record Type | What It Does | Example |
|---|---|---|
| **A** | Maps domain to IPv4 address | `example.com → 93.184.216.34` |
| **AAAA** | Maps domain to IPv6 address | `example.com → 2606:2800:220:1:...` |
| **CNAME** | Maps domain to another domain (alias) | `www.example.com → example.com` |
| **MX** | Directs email to mail servers | `example.com → mail.example.com` |
| **TXT** | Stores text (used for verification, SPF) | `example.com → "v=spf1 ..."` |
| **NS** | Specifies authoritative name servers | `example.com → ns1.awsdns.com` |

### Cloud DNS Services

| Provider | Service | Purpose |
|---|---|---|
| AWS | **Route 53** | DNS hosting, domain registration, health checks, routing policies |
| Azure | **Azure DNS** | DNS hosting, integrates with Azure resources |
| GCP | **Cloud DNS** | DNS hosting, DNSSEC support |

---

## Load Balancers

### What Is a Load Balancer?

A **load balancer** distributes incoming traffic across **multiple servers** so that no single server gets overwhelmed.

### Why Use One?

- **High availability** — If one server dies, traffic goes to healthy ones.
- **Scalability** — Handle more traffic by adding more servers behind the load balancer.
- **Performance** — Spread the load evenly.

### Types of Load Balancers

| Type | OSI Layer | Routes Based On | Use Case |
|---|---|---|---|
| **Application Load Balancer (ALB)** | Layer 7 (HTTP/HTTPS) | URL path, headers, hostname | Web apps, microservices, API routing |
| **Network Load Balancer (NLB)** | Layer 4 (TCP/UDP) | IP address, port | High-performance, low-latency apps |
| **Gateway Load Balancer (GLB)** | Layer 3 (Network) | All traffic | Firewalls, intrusion detection appliances |

### Health Checks

Load balancers regularly send **health check** requests to backend servers:
- If a server responds → **Healthy** → Keep sending traffic.
- If a server doesn't respond → **Unhealthy** → Stop sending traffic to it.

```
User Request
   ↓
Load Balancer
   ├── Server 1 ✅ (healthy → receives traffic)
   ├── Server 2 ✅ (healthy → receives traffic)
   └── Server 3 ❌ (unhealthy → no traffic)
```

---

## CDN (Content Delivery Network)

### What Is a CDN?

A **CDN** is a network of servers distributed around the world that **cache and deliver content** (images, videos, CSS, JavaScript) from a location **close to the user**.

### Why Use a CDN?

- **Faster page loads** — Content served from a nearby server, not your origin server across the world.
- **Reduced load on origin** — The CDN handles most requests.
- **DDoS protection** — CDNs absorb malicious traffic before it reaches your server.

### How It Works

```
Without CDN:
User in Tokyo → Request travels to origin server in Virginia → Slow (high latency)

With CDN:
User in Tokyo → Request goes to CDN edge server in Tokyo → Fast (low latency)
```

### Cloud CDN Services

| Provider | Service |
|---|---|
| AWS | **CloudFront** |
| Azure | **Azure CDN** / **Azure Front Door** |
| GCP | **Cloud CDN** |
| Independent | **Cloudflare**, **Akamai**, **Fastly** |

---

## VPC/VNet Peering

### What Is It?

**VPC Peering** (AWS) or **VNet Peering** (Azure) connects two separate VPCs/VNets so they can communicate using **private IP addresses** — as if they were on the same network.

### When to Use It

- You have separate VPCs for different environments (dev, staging, production)
- You need resources in different VPCs to communicate privately
- You want to share services (like a centralised logging VPC) across multiple VPCs

### Key Points

- Traffic stays on the **provider's private network** (does not go over the public internet).
- Peering is **non-transitive** — If VPC-A peers with VPC-B, and VPC-B peers with VPC-C, VPC-A **cannot** automatically talk to VPC-C. You need a separate peering connection.
- VPC CIDR ranges **must not overlap**.

```
VPC-A (10.0.0.0/16) ←→ Peering ←→ VPC-B (10.1.0.0/16)  ✅ Works
VPC-A (10.0.0.0/16) ←→ Peering ←→ VPC-C (10.0.0.0/16)  ❌ CIDR overlap!
```

---

## Common Cloud Networking Architecture

### The 3-Tier Architecture

This is the most common pattern for deploying web applications in the cloud:

```
┌─────────────────────────────────────────────────────────┐
│                         INTERNET                         │
└─────────────────────┬───────────────────────────────────┘
                      │
              ┌───────▼────────┐
              │ Internet Gateway│
              └───────┬────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│  PUBLIC SUBNET (Tier 1 — Presentation)                   │
│                                                          │
│  ┌──────────────────┐                                    │
│  │  Load Balancer    │  ← Distributes incoming traffic   │
│  └──────────┬───────┘                                    │
│             │                                            │
│  ┌──────────▼───────┐  ┌───────────────┐                │
│  │  Web Server 1     │  │  Web Server 2  │  ← Auto-scaled│
│  └──────────────────┘  └───────────────┘                │
└─────────────────────┬───────────────────────────────────┘
                      │ (Security Group allows only
                      │  traffic from web servers)
┌─────────────────────▼───────────────────────────────────┐
│  PRIVATE SUBNET (Tier 2 — Application/Logic)             │
│                                                          │
│  ┌──────────────────┐  ┌───────────────┐                │
│  │  App Server 1     │  │  App Server 2  │               │
│  └──────────────────┘  └───────────────┘                │
│                                                          │
│  ┌──────────────────┐                                    │
│  │  NAT Gateway      │  ← Outbound internet only         │
│  └──────────────────┘                                    │
└─────────────────────┬───────────────────────────────────┘
                      │ (Security Group allows only
                      │  traffic from app servers)
┌─────────────────────▼───────────────────────────────────┐
│  DATA SUBNET (Tier 3 — Database)                         │
│                                                          │
│  ┌──────────────────┐  ┌───────────────┐                │
│  │  Database Primary │  │  Database     │  ← Multi-AZ    │
│  │  (AZ-1)          │  │  Standby (AZ-2)│     failover   │
│  └──────────────────┘  └───────────────┘                │
│                                                          │
│  NO internet access — completely isolated                │
└─────────────────────────────────────────────────────────┘
```

### Why This Pattern Works

| Tier | Subnet Type | Why |
|---|---|---|
| Web servers | Public | Need to receive traffic from the internet |
| App servers | Private | Only web servers should talk to them |
| Databases | Private (isolated) | Only app servers should talk to them |

> [!tip] Security Principle
> Each tier can ONLY communicate with the tier directly above or below it. The internet can NEVER reach the database directly.

---

## Networking Troubleshooting Checklist

When something "can't connect" in the cloud, check these in order:

1. ✅ **Security Group** — Is the port open for inbound traffic?
2. ✅ **Network ACL** — Is the subnet-level firewall allowing traffic?
3. ✅ **Route Table** — Does the subnet have a route to the destination?
4. ✅ **Internet Gateway** — Is one attached to the VPC? (for internet access)
5. ✅ **NAT Gateway** — Does the private subnet have a NAT Gateway? (for outbound internet)
6. ✅ **Public IP** — Does the instance have a public or Elastic IP? (for inbound internet)
7. ✅ **OS Firewall** — Is the firewall inside the VM (iptables, Windows Firewall) blocking traffic?
8. ✅ **DNS** — Can the resource resolve the hostname?

---

## What's Next?

- Provider-specific networking: [[AWS Networking]] / [[Azure Networking]]
- Security concepts: [[Cloud Security Fundamentals]]
- Back to basics: [[Cloud Fundamentals]]

---

> [!success] Summary
> Cloud networking is built around VPCs (isolated networks), subnets (public and private), gateways (for internet and VPN access), route tables (traffic direction), and firewalls (security groups and NACLs). A well-designed network uses the 3-tier architecture pattern to isolate web, application, and database layers for maximum security and reliability.
