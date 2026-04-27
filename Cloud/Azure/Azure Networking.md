# Azure Networking

> **Networking is the backbone of every cloud deployment.** Understanding Azure
> networking is essential — it controls how your resources communicate with each
> other, the internet, and your on-premises infrastructure.

---

## Table of Contents

- [[#Azure Virtual Network (VNet)]]
- [[#Subnets]]
- [[#Network Security Groups (NSGs)]]
- [[#Public IP Addresses]]
- [[#Azure DNS]]
- [[#Azure Load Balancer]]
- [[#Azure Application Gateway]]
- [[#Azure Bastion]]
- [[#VNet Peering]]
- [[#Azure VPN Gateway]]
- [[#Azure ExpressRoute]]
- [[#NSG vs AWS Security Groups]]
- [[#Typical Azure Network Architecture]]

---

## Azure Virtual Network (VNet)

**Azure Virtual Network (VNet)** is your **private, isolated network** in Azure.
It's the foundation of Azure networking — like a **virtual data center** in the cloud.

> **AWS equivalent**: VNet ≈ **VPC** (Virtual Private Cloud)

### Why VNets Matter

- Every Azure resource that needs networking (VMs, databases, App Services with
  VNet integration, AKS clusters) **must be in a VNet**.
- VNets are **region-scoped** — a VNet exists in one Azure region.
- VNets are **free** — you pay for the resources inside them, not the VNet itself.
- By default, resources in a VNet **can communicate** with each other.
- By default, VNets are **isolated** from each other (no cross-VNet traffic).

### Key Concepts

| Concept          | Description                                                   |
|------------------|---------------------------------------------------------------|
| **Address Space** | The IP range for your VNet (e.g., `10.0.0.0/16` = 65,536 IPs)|
| **Subnet**       | A subdivision of the VNet (e.g., `10.0.1.0/24` = 256 IPs)    |
| **NIC**          | Network Interface Card — attaches a VM to a subnet            |
| **Private IP**   | Internal IP within the VNet (auto-assigned or static)         |
| **Public IP**    | Internet-facing IP (optional, assigned to NIC or LB)          |

### CIDR Notation Refresher

If CIDR notation is new to you:

| CIDR           | Total IPs | Usable IPs* | Use Case               |
|----------------|-----------|-------------|------------------------|
| `/16`          | 65,536    | 65,531      | Entire VNet            |
| `/24`          | 256       | 251         | Typical subnet         |
| `/27`          | 32        | 27          | Small subnet           |
| `/28`          | 16        | 11          | Minimum useful subnet  |

> *Azure reserves 5 IPs per subnet (first, last, and 3 for internal services).

### Hands-on: Create a VNet

```bash
# Step 1: Create a VNet with one subnet
# Address space: 10.0.0.0/16 (65,536 IPs for the whole VNet)
# First subnet: 10.0.1.0/24 (256 IPs for public-facing resources)
az network vnet create \
  --resource-group myResourceGroup \
  --name myVNet \
  --address-prefix 10.0.0.0/16 \
  --subnet-name PublicSubnet \
  --subnet-prefix 10.0.1.0/24
```

```bash
# Step 2: Add a second subnet for private/backend resources
az network vnet subnet create \
  --resource-group myResourceGroup \
  --vnet-name myVNet \
  --name PrivateSubnet \
  --address-prefix 10.0.2.0/24
```

```bash
# Step 3: Verify — list all subnets
az network vnet subnet list \
  --resource-group myResourceGroup \
  --vnet-name myVNet \
  --output table
```

```bash
# View VNet details
az network vnet show \
  --resource-group myResourceGroup \
  --name myVNet \
  --output table
```

```bash
# List all VNets in your resource group
az network vnet list --resource-group myResourceGroup --output table
```

---

## Subnets

**Subnets** divide your VNet into smaller, logical segments. Each subnet gets
a portion of the VNet's address space.

### Why Subnets?

- **Organize resources** by role (web tier, app tier, data tier)
- **Apply different security rules** (NSGs) per subnet
- **Control routing** — different subnets can have different route tables
- **Isolate workloads** — a compromised web server can't directly reach the database

### Types of Subnets

#### Public Subnet
- Has a route to the internet (via default route or NAT Gateway)
- Hosts resources that need internet access: load balancers, web servers, bastion
- Resources with Public IPs are directly accessible from the internet

#### Private Subnet
- **No direct internet access** (no public IPs assigned)
- Hosts backend services: application servers, databases, internal APIs
- Can reach the internet via **NAT Gateway** (for outbound only — patches, API calls)

#### Dedicated / Reserved Subnets

Some Azure services **require** their own dedicated subnet:

| Subnet Name              | Required By                  | Purpose                      |
|--------------------------|------------------------------|------------------------------|
| `GatewaySubnet`          | VPN Gateway, ExpressRoute    | Must be named exactly this   |
| `AzureBastionSubnet`     | Azure Bastion                | Must be /26 or larger        |
| `AzureFirewallSubnet`    | Azure Firewall               | Must be /26 or larger        |
| `AzureFirewallManagementSubnet` | Azure Firewall (forced tunnel) | Management traffic  |

> ⚠️ **Important**: These subnet names are **exact** — Azure won't work if you
> name them differently.

### Hands-on: Subnet for a Database Tier

```bash
# Add a database subnet
az network vnet subnet create \
  --resource-group myResourceGroup \
  --vnet-name myVNet \
  --name DatabaseSubnet \
  --address-prefix 10.0.3.0/24
```

```bash
# You can also add a subnet for Azure Bastion
az network vnet subnet create \
  --resource-group myResourceGroup \
  --vnet-name myVNet \
  --name AzureBastionSubnet \
  --address-prefix 10.0.4.0/26
```

---

## Network Security Groups (NSGs)

**Network Security Groups (NSGs)** are Azure's built-in firewall. They contain
**security rules** that allow or deny network traffic.

> **AWS equivalent**: NSG ≈ **Security Groups + NACLs** combined

### Where NSGs Can Be Applied

NSGs can be attached to:
1. **Subnets** — rules apply to ALL resources in that subnet
2. **Network Interface Cards (NICs)** — rules apply to a specific VM

> **Best practice**: Apply NSGs at the **subnet level** for broad rules,
> and at the NIC level for resource-specific exceptions.

### How NSG Rules Work

Each rule has these properties:

| Property        | Description                                     | Example         |
|-----------------|-------------------------------------------------|-----------------|
| **Priority**    | 100-4096. Lower number = evaluated first        | 100             |
| **Direction**   | Inbound or Outbound                             | Inbound         |
| **Action**      | Allow or Deny                                   | Allow           |
| **Source**       | IP, CIDR, Service Tag, or ASG                  | 10.0.0.0/16     |
| **Destination** | IP, CIDR, Service Tag, or ASG                   | *               |
| **Port**        | Single port, range, or * (all)                  | 80, 443, 22     |
| **Protocol**    | TCP, UDP, ICMP, or * (any)                      | Tcp             |

### Rule Evaluation Order

1. Rules are evaluated by **priority** (lowest number first).
2. Once a matching rule is found, it's applied — no further rules are checked.
3. If no custom rule matches, **default rules** apply.

### Default NSG Rules (built-in, cannot delete)

| Priority | Name                        | Direction | Action | Description                          |
|----------|-----------------------------|-----------|--------|--------------------------------------|
| 65000    | AllowVnetInBound            | Inbound   | Allow  | Allow traffic within the VNet        |
| 65001    | AllowAzureLoadBalancerInBound | Inbound | Allow  | Allow Azure Load Balancer probes     |
| 65500    | DenyAllInBound              | Inbound   | Deny   | Deny everything else                 |
| 65000    | AllowVnetOutBound           | Outbound  | Allow  | Allow traffic within the VNet        |
| 65001    | AllowInternetOutBound       | Outbound  | Allow  | Allow outbound to internet           |
| 65500    | DenyAllOutBound             | Outbound  | Deny   | Deny everything else                 |

> **Key insight**: By default, VNet-internal traffic is allowed, internet
> outbound is allowed, but ALL inbound from internet is **denied**.

### Hands-on: Create and Configure an NSG

```bash
# Step 1: Create an NSG
az network nsg create \
  --resource-group myResourceGroup \
  --name myNSG
```

```bash
# Step 2: Allow SSH (port 22) — for Linux VMs
az network nsg rule create \
  --resource-group myResourceGroup \
  --nsg-name myNSG \
  --name AllowSSH \
  --priority 100 \
  --destination-port-ranges 22 \
  --protocol Tcp \
  --access Allow \
  --direction Inbound
```

```bash
# Step 3: Allow HTTP (port 80) — for web servers
az network nsg rule create \
  --resource-group myResourceGroup \
  --nsg-name myNSG \
  --name AllowHTTP \
  --priority 200 \
  --destination-port-ranges 80 \
  --protocol Tcp \
  --access Allow \
  --direction Inbound
```

```bash
# Step 4: Allow HTTPS (port 443)
az network nsg rule create \
  --resource-group myResourceGroup \
  --nsg-name myNSG \
  --name AllowHTTPS \
  --priority 300 \
  --destination-port-ranges 443 \
  --protocol Tcp \
  --access Allow \
  --direction Inbound
```

```bash
# Step 5: Deny all other inbound (explicit — good practice)
az network nsg rule create \
  --resource-group myResourceGroup \
  --nsg-name myNSG \
  --name DenyAllInbound \
  --priority 4096 \
  --destination-port-ranges '*' \
  --protocol '*' \
  --access Deny \
  --direction Inbound
```

```bash
# Step 6: Associate the NSG with a subnet
az network vnet subnet update \
  --resource-group myResourceGroup \
  --vnet-name myVNet \
  --name PublicSubnet \
  --network-security-group myNSG
```

```bash
# Verify: List all rules in the NSG
az network nsg rule list \
  --resource-group myResourceGroup \
  --nsg-name myNSG \
  --output table
```

```bash
# View effective security rules for a specific NIC
# az network nic list-effective-nsg --resource-group myResourceGroup --name myNIC
```

### Service Tags — Simplify NSG Rules

Instead of specifying IP ranges, use **Service Tags** — predefined groups of IPs:

| Service Tag       | Represents                              |
|-------------------|-----------------------------------------|
| `Internet`        | All public internet IPs                 |
| `VirtualNetwork`  | VNet address space + peered VNets       |
| `AzureLoadBalancer` | Azure's health probes                 |
| `Storage`         | Azure Storage service IPs               |
| `Sql`             | Azure SQL Database service IPs          |
| `AzureCloud`      | All Azure datacenter IPs               |

---

## Public IP Addresses

**Public IP addresses** allow Azure resources to be reachable from the internet.

### SKU Comparison

| Feature          | Basic SKU              | Standard SKU                     |
|------------------|------------------------|----------------------------------|
| **Allocation**   | Dynamic or Static      | Static only                      |
| **Zone Redundancy** | No                  | Yes (survives zone failure)      |
| **Availability** | No SLA                 | 99.99% SLA                      |
| **Security**     | Open by default        | Closed by default (needs NSG)   |
| **Load Balancer**| Basic LB only          | Standard LB                      |

> **Always use Standard SKU** for production. Basic is being deprecated.

### Hands-on: Create a Public IP

```bash
# Create a static, zone-redundant public IP
az network public-ip create \
  --resource-group myResourceGroup \
  --name myPublicIP \
  --sku Standard \
  --allocation-method Static
```

```bash
# View the assigned IP address
az network public-ip show \
  --resource-group myResourceGroup \
  --name myPublicIP \
  --query ipAddress \
  --output tsv
```

```bash
# List all public IPs in your resource group
az network public-ip list --resource-group myResourceGroup --output table
```

> ⚠️ **Unused public IPs cost money!** Always delete public IPs you're not using.
> See [[Azure Billing]] for cost optimization tips.

---

## Azure DNS

**Azure DNS** lets you host your DNS zones in Azure and manage DNS records.

> DNS translates human-readable names (www.example.com) to IP addresses (1.2.3.4).

### Hands-on: Host a DNS Zone

```bash
# Step 1: Create a DNS zone
az network dns zone create \
  --resource-group myResourceGroup \
  --name example.com
```

```bash
# Step 2: Add an A record (maps name → IP)
az network dns record-set a add-record \
  --resource-group myResourceGroup \
  --zone-name example.com \
  --record-set-name www \
  --ipv4-address 1.2.3.4
```

```bash
# Step 3: Add a CNAME record (maps name → another name)
az network dns record-set cname set-record \
  --resource-group myResourceGroup \
  --zone-name example.com \
  --record-set-name blog \
  --cname blog.azurewebsites.net
```

```bash
# View all records in the zone
az network dns record-set list \
  --resource-group myResourceGroup \
  --zone-name example.com \
  --output table
```

```bash
# Get nameservers (you'll point your domain registrar to these)
az network dns zone show \
  --resource-group myResourceGroup \
  --name example.com \
  --query nameServers
```

### Azure Private DNS Zones

For internal name resolution within your VNet (not public internet):

```bash
# Create a private DNS zone
az network private-dns zone create \
  --resource-group myResourceGroup \
  --name internal.mycompany.com

# Link it to a VNet
az network private-dns link vnet create \
  --resource-group myResourceGroup \
  --zone-name internal.mycompany.com \
  --name myVNetLink \
  --virtual-network myVNet \
  --registration-enabled true
```

---

## Azure Load Balancer

**Azure Load Balancer** operates at **Layer 4 (TCP/UDP)**. It distributes
incoming network traffic across multiple backend instances (VMs).

> **AWS equivalent**: Azure Load Balancer ≈ **Network Load Balancer (NLB)**

### Key Concepts

| Concept              | Description                                        |
|----------------------|----------------------------------------------------|
| **Frontend IP**      | The public IP that clients connect to              |
| **Backend Pool**     | The group of VMs receiving traffic                  |
| **Health Probe**     | Checks if backend VMs are healthy                   |
| **Load Balancing Rule** | Maps frontend port to backend port              |

### SKU Comparison

| Feature          | Basic                  | Standard                          |
|------------------|------------------------|-----------------------------------|
| **Backend Pool** | Up to 300 instances    | Up to 1000 instances              |
| **Health Probes**| TCP, HTTP              | TCP, HTTP, HTTPS                  |
| **Availability Zones** | No                | Yes (zone-redundant)              |
| **SLA**          | No SLA                 | 99.99%                            |
| **Security**     | Open by default        | Closed by default (needs NSG)     |

### Hands-on: Create a Load Balancer

```bash
# Step 1: Create a Standard Load Balancer
az network lb create \
  --resource-group myResourceGroup \
  --name myLoadBalancer \
  --sku Standard \
  --frontend-ip-name myFrontEnd \
  --backend-pool-name myBackEndPool \
  --public-ip-address myPublicIP
```

```bash
# Step 2: Create a health probe
az network lb probe create \
  --resource-group myResourceGroup \
  --lb-name myLoadBalancer \
  --name myHealthProbe \
  --protocol tcp \
  --port 80
```

```bash
# Step 3: Create a load balancing rule
az network lb rule create \
  --resource-group myResourceGroup \
  --lb-name myLoadBalancer \
  --name myHTTPRule \
  --protocol tcp \
  --frontend-port 80 \
  --backend-port 80 \
  --frontend-ip-name myFrontEnd \
  --backend-pool-name myBackEndPool \
  --probe-name myHealthProbe
```

---

## Azure Application Gateway

**Azure Application Gateway** is a **Layer 7 (HTTP/HTTPS)** load balancer with
advanced features.

> **AWS equivalent**: Application Gateway ≈ **ALB (Application Load Balancer) + WAF**

### Features

| Feature                  | Description                                          |
|--------------------------|------------------------------------------------------|
| **URL-based routing**    | Route /api/* to backend A, /images/* to backend B    |
| **SSL termination**      | Offload SSL/TLS decryption from your backend servers |
| **Cookie-based affinity**| Sticky sessions — same user goes to same backend     |
| **Web Application Firewall (WAF)** | Protects against SQL injection, XSS, etc.  |
| **Autoscaling**          | Scale in/out based on traffic                        |
| **Multi-site hosting**   | Host multiple websites on one Application Gateway    |

### When to Use Which

| Scenario                          | Use                     |
|-----------------------------------|-------------------------|
| Simple TCP/UDP distribution       | Azure Load Balancer     |
| HTTP routing, SSL, WAF            | Application Gateway     |
| Global traffic management         | Azure Front Door        |
| DNS-based routing                 | Traffic Manager         |

---

## Azure Bastion

**Azure Bastion** provides **secure RDP/SSH access** to your VMs **without
exposing them to the internet**. No public IPs needed on your VMs.

### How It Works

```
You (browser) → Azure Portal → Azure Bastion → Private IP of VM
```

- No need for a public IP on the VM
- No need to open RDP (3389) or SSH (22) to the internet
- Connection happens through the Azure Portal (HTML5-based)
- Bastion is deployed in its own subnet (`AzureBastionSubnet`, /26 or larger)

### Why Use Bastion

| Without Bastion                          | With Bastion                             |
|------------------------------------------|------------------------------------------|
| VM needs a public IP                     | VM stays fully private                   |
| NSG must allow SSH/RDP from internet     | No internet-facing ports needed          |
| Vulnerable to brute-force attacks        | Portal-based access, more secure         |
| Need VPN for private access              | Just use the Azure Portal                |

### Hands-on: Create Azure Bastion

```bash
# Prerequisite: AzureBastionSubnet must exist (created earlier)

# Create Bastion
az network bastion create \
  --resource-group myResourceGroup \
  --name myBastion \
  --vnet-name myVNet \
  --public-ip-address myBastionPublicIP
```

> Then in the Azure Portal: Go to your VM → Connect → Bastion → Enter credentials.

---

## VNet Peering

**VNet Peering** connects two VNets so resources in each can communicate directly.

> Traffic between peered VNets uses the **Azure backbone network** — it never
> goes over the public internet. Low latency, high bandwidth.

### Types

| Type                  | Description                                       |
|-----------------------|---------------------------------------------------|
| **Regional Peering**  | Connect VNets in the same Azure region             |
| **Global Peering**    | Connect VNets in different Azure regions           |

### Key Properties

- **Non-transitive**: If VNet A is peered with VNet B, and VNet B is peered
  with VNet C, VNet A **cannot** reach VNet C automatically.
- **Both sides must peer**: You create a peering from A→B AND from B→A.
- **No IP overlap**: The address spaces of peered VNets cannot overlap.

### Hands-on: Peer Two VNets

```bash
# Create peering from VNet1 to VNet2
az network vnet peering create \
  --resource-group myResourceGroup \
  --name VNet1ToVNet2 \
  --vnet-name myVNet1 \
  --remote-vnet myVNet2 \
  --allow-vnet-access
```

```bash
# Create peering from VNet2 to VNet1 (both directions required!)
az network vnet peering create \
  --resource-group myResourceGroup \
  --name VNet2ToVNet1 \
  --vnet-name myVNet2 \
  --remote-vnet myVNet1 \
  --allow-vnet-access
```

```bash
# Check peering status
az network vnet peering list \
  --resource-group myResourceGroup \
  --vnet-name myVNet1 \
  --output table
```

---

## Azure VPN Gateway

**Azure VPN Gateway** connects your **on-premises network to Azure** over an
encrypted tunnel through the public internet.

### Types

| Type               | Description                                          | Use Case                    |
|--------------------|------------------------------------------------------|-----------------------------|
| **Site-to-Site (S2S)** | Connects your office/datacenter to Azure VNet    | Main office → Azure         |
| **Point-to-Site (P2S)** | Connects individual computers to Azure VNet     | Remote workers → Azure      |
| **VNet-to-VNet**   | Connects two Azure VNets via VPN                     | Cross-region encryption     |

### Key Points

- Requires a dedicated `GatewaySubnet` in your VNet.
- Provisioning takes 30-45 minutes.
- SKUs determine bandwidth: Basic (100 Mbps) to VpnGw5 (10 Gbps).
- Uses IPsec/IKE protocols for encryption.

```bash
# Create a VPN Gateway (takes 30-45 minutes!)
az network vnet-gateway create \
  --resource-group myResourceGroup \
  --name myVPNGateway \
  --vnet myVNet \
  --gateway-type Vpn \
  --vpn-type RouteBased \
  --sku VpnGw1 \
  --public-ip-address myVPNGatewayIP \
  --no-wait
```

---

## Azure ExpressRoute

**Azure ExpressRoute** provides a **dedicated, private connection** between your
on-premises infrastructure and Azure. Traffic does **not** go over the internet.

### ExpressRoute vs VPN Gateway

| Feature          | VPN Gateway                   | ExpressRoute                     |
|------------------|-------------------------------|----------------------------------|
| **Connection**   | Over public internet          | Dedicated private line           |
| **Encryption**   | IPsec (built-in)              | Not encrypted by default*        |
| **Bandwidth**    | Up to 10 Gbps                 | Up to 100 Gbps                   |
| **Latency**      | Variable (internet)           | Predictable, low                 |
| **Cost**          | Low                           | High (monthly circuit fee)       |
| **Redundancy**   | Active-active possible        | Built-in redundant connections   |
| **Best For**     | Small offices, dev/test       | Enterprises, compliance, large data |

> *You can add VPN encryption on top of ExpressRoute for extra security.

### When to Use ExpressRoute

- **Regulatory compliance** — data must not traverse the public internet
- **High bandwidth** — moving large datasets (backups, replication)
- **Consistent latency** — real-time financial trading, video streaming
- **Enterprise** — connecting large datacenters to Azure

---

## NSG vs AWS Security Groups

If you're coming from AWS, this comparison will help:

| Feature                | Azure NSG                           | AWS Security Groups              |
|------------------------|-------------------------------------|----------------------------------|
| **Attached To**        | Subnet or NIC                       | Instance (ENI)                   |
| **Deny Rules**         | ✅ Yes (Allow + Deny)               | ❌ No (Allow only)              |
| **Priority System**    | ✅ Yes (100-4096)                   | ❌ No (all rules evaluated)     |
| **Stateful**           | ✅ Yes                              | ✅ Yes                          |
| **Default Inbound**    | Deny all (except VNet + LB)        | Deny all                        |
| **Default Outbound**   | Allow VNet + Internet               | Allow all                       |
| **Service Tags**       | ✅ Yes (e.g., `Storage`, `Sql`)    | ❌ No (use prefix lists)        |
| **Applies to Subnet**  | ✅ Yes                              | ❌ No (SG is per-instance)      |

### The AWS NACL Comparison

| Feature          | Azure NSG              | AWS NACL                   | AWS Security Group      |
|------------------|------------------------|----------------------------|-------------------------|
| **Level**        | Subnet + NIC           | Subnet                     | Instance                |
| **Stateful**     | Yes                    | No (stateless)             | Yes                     |
| **Deny Rules**   | Yes                    | Yes                        | No                      |
| **Priority**     | Yes                    | Yes                        | No                      |

> **Key takeaway**: Azure NSG combines the best of AWS Security Groups (stateful)
> and NACLs (deny rules, subnet-level, priority-based) into a single construct.

---

## Typical Azure Network Architecture

Here's a reference architecture for a typical web application:

```
                    Internet
                       │
                 ┌─────▼─────┐
                 │ Public IP  │
                 │ (Standard) │
                 └─────┬─────┘
                       │
              ┌────────▼────────┐
              │  Application    │    ← Layer 7 Load Balancer
              │  Gateway + WAF  │      URL routing, SSL termination
              └────────┬────────┘
                       │
           ┌───────────▼───────────┐
           │     VNet: 10.0.0.0/16 │
           │                       │
           │  ┌──────────────────┐ │
           │  │ PublicSubnet     │ │  ← NSG: Allow 80, 443
           │  │ 10.0.1.0/24     │ │    App Gateway lives here
           │  └────────┬─────────┘ │
           │           │           │
           │  ┌────────▼─────────┐ │
           │  │ AppSubnet        │ │  ← NSG: Allow from PublicSubnet only
           │  │ 10.0.2.0/24     │ │    VMs / AKS / App Services
           │  └────────┬─────────┘ │
           │           │           │
           │  ┌────────▼─────────┐ │
           │  │ DatabaseSubnet   │ │  ← NSG: Allow from AppSubnet only
           │  │ 10.0.3.0/24     │ │    Azure SQL, Cosmos DB endpoints
           │  └──────────────────┘ │
           │                       │
           │  ┌──────────────────┐ │
           │  │ AzureBastionSubnet│ │  ← For secure VM management
           │  │ 10.0.4.0/26     │ │
           │  └──────────────────┘ │
           └───────────────────────┘
```

### Architecture Principles

1. **Defence in depth** — NSGs on every subnet, not just the edge.
2. **Least privilege** — Each subnet's NSG only allows what's needed.
3. **No public IPs on VMs** — Use Azure Bastion for management access.
4. **Application Gateway** at the edge — WAF protection, SSL offloading.
5. **Database subnet** — Only accessible from the application tier.
6. **Private Endpoints** — Connect to managed services (SQL, Storage) via
   private IPs inside your VNet instead of public endpoints.

### Private Endpoints — Connect Securely to Managed Services

```bash
# Create a private endpoint for Azure SQL
az network private-endpoint create \
  --resource-group myResourceGroup \
  --name mySQL-PrivateEndpoint \
  --vnet-name myVNet \
  --subnet DatabaseSubnet \
  --private-connection-resource-id <sql-server-resource-id> \
  --group-id sqlServer \
  --connection-name mySQLConnection
```

> Private Endpoints give your managed service a **private IP** inside your VNet.
> Traffic stays entirely on the Azure backbone — never touches the internet.

---

## Networking Cleanup

```bash
# Delete resources when done learning (in reverse order of dependencies)
az network bastion delete --resource-group myResourceGroup --name myBastion
az network lb delete --resource-group myResourceGroup --name myLoadBalancer
az network public-ip delete --resource-group myResourceGroup --name myPublicIP
az network nsg delete --resource-group myResourceGroup --name myNSG
az network vnet delete --resource-group myResourceGroup --name myVNet

# Or just delete the entire resource group (deletes everything inside)
az group delete --name myResourceGroup --yes --no-wait
```

---

## Summary

| Service                | Layer | Purpose                                   | AWS Equivalent    |
|------------------------|-------|-------------------------------------------|-------------------|
| **VNet**               | —     | Isolated network                          | VPC               |
| **Subnet**             | —     | Segment within VNet                       | Subnet            |
| **NSG**                | 3-4   | Firewall rules                            | SG + NACL         |
| **Public IP**          | —     | Internet-facing IP address                | Elastic IP        |
| **Azure DNS**          | —     | DNS hosting                               | Route 53          |
| **Load Balancer**      | 4     | TCP/UDP traffic distribution              | NLB               |
| **Application Gateway**| 7     | HTTP/HTTPS routing + WAF                  | ALB + WAF         |
| **Azure Bastion**      | —     | Secure RDP/SSH without public IPs         | Session Manager   |
| **VNet Peering**       | —     | Connect VNets                             | VPC Peering       |
| **VPN Gateway**        | —     | Encrypted tunnel to on-premises           | VPN Gateway       |
| **ExpressRoute**       | —     | Dedicated private connection              | Direct Connect    |

---

## Related Notes

- [[Cloud Networking Basics]] — General networking concepts (TCP/IP, DNS, CIDR)
- [[Azure Getting Started]] — Setting up your Azure account and CLI
- [[Azure Compute]] — VMs, App Service — the resources that live in your VNets
- [[Azure Databases]] — Database services that connect to your network
- [[Azure Billing]] — Network egress costs and cost optimization
- [[AWS Networking]] — Compare with AWS VPC, Security Groups, NLB/ALB
