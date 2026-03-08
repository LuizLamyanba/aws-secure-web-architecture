# Network Design — VPC Architecture for Secure Web Application

## Overview

This document describes the network architecture underpinning the secure web application deployment on AWS. The design uses **Amazon VPC** to establish strict network-level isolation, ensuring backend compute resources have no direct exposure to the public internet.

Every routing and firewall decision in this architecture follows the **principle of least-privilege networking** — resources are granted only the connectivity they need, nothing more.

---

## VPC Configuration

| Field | Value |
|---|---|
| **VPC Name** | `secure-project-vpc` |
| **CIDR Block** | `10.0.0.0/16` |
| **Region** | `ap-south-1` (Mumbai) |
| **Tenancy** | Default |
| **Available IPs** | 65,536 |

The `/16` CIDR was chosen to provide ample address space for subnet expansion — current subnets consume a small fraction of this range, leaving room for additional tiers (e.g., database subnet, management subnet) without requiring re-addressing.

---

## Subnet Architecture

The VPC is divided into three subnets across two Availability Zones, following a **public/private tier model**.

```
VPC: 10.0.0.0/16
│
├── public-subnet-1a    10.0.1.0/24    ap-south-1a   ← ALB node
├── public-subnet-1b    10.0.2.0/24    ap-south-1b   ← ALB node + NAT Gateway
│
└── private-subnet-1a   10.0.3.0/24    ap-south-1a   ← EC2 (Nginx)
```

### Subnet Role Summary

| Subnet | CIDR | AZ | Hosts | Internet Access |
|---|---|---|---|---|
| `public-subnet-1a` | `10.0.1.0/24` | `ap-south-1a` | ALB | Inbound + Outbound via IGW |
| `public-subnet-1b` | `10.0.2.0/24` | `ap-south-1b` | ALB, NAT Gateway | Inbound + Outbound via IGW |
| `private-subnet-1a` | `10.0.3.0/24` | `ap-south-1a` | EC2 (Nginx) | Outbound only via NAT |

Each `/24` subnet provides 251 usable IP addresses — sufficient for the current workload with room for horizontal scaling.

---

## Public Subnets

Public subnets host only **internet-facing, stateless** infrastructure. No application logic or sensitive workloads run here.

**Resources in public subnets:**

- **Application Load Balancer** — spans both public subnets across two AZs for high availability
- **NAT Gateway** — deployed in `public-subnet-1b`, provides outbound-only internet access for the private subnet

Instances in public subnets may be assigned public IP addresses. However, Security Group rules (detailed below) restrict what can actually reach them.

**Why two public subnets across two AZs?**

The ALB requires at least two subnets in separate AZs to operate. This ensures the load balancer remains available even if one AZ experiences an outage — a standard requirement for any production-grade ALB deployment.

---

## Private Subnet

The backend EC2 instance running Nginx is deployed exclusively in `private-subnet-1a`.

**Key properties of the private subnet:**

- No route to the Internet Gateway — inbound connections from the internet are impossible at the routing layer
- No public IP address assigned to instances
- Outbound internet access permitted only through the NAT Gateway (for OS updates, AWS API calls)
- Reachable internally only from resources within the VPC, subject to Security Group rules

This means even if the application itself were vulnerable, an attacker on the internet would have no network path to reach the EC2 instance directly. The origin is unreachable by design.

---

## Internet Gateway

An **Internet Gateway (IGW)** is attached to the VPC to enable bidirectional internet connectivity for public subnet resources.

| Field | Value |
|---|---|
| **Attached to** | `secure-project-vpc` |
| **Used by** | Public subnets (via route table) |
| **Enables** | Inbound traffic to ALB, outbound traffic from public resources |

The IGW is not accessible from the private subnet — its route is only present in the public route table.

---

## NAT Gateway

A **NAT Gateway** is deployed in `public-subnet-1b` to provide the private EC2 instance with controlled outbound internet access.

| Field | Value |
|---|---|
| **Deployed in** | `public-subnet-1b` (public subnet) |
| **Elastic IP** | Yes (static public IP for outbound traffic) |
| **Direction** | Outbound only — internet cannot initiate connections inbound |

**Outbound traffic flow from EC2:**

```
EC2 (private-subnet-1a)
      │
      ▼  [private route table: 0.0.0.0/0 → NAT Gateway]
NAT Gateway (public-subnet-1b)
      │
      ▼  [public route table: 0.0.0.0/0 → Internet Gateway]
Internet Gateway
      │
      ▼
Internet
```

**Use cases for outbound access:**

- OS package updates (`yum update`)
- AWS SSM Agent communication with Systems Manager endpoints
- Downloading application dependencies

**What the NAT Gateway does NOT allow:**

The internet cannot initiate a connection to the EC2 instance through the NAT Gateway. NAT translation is one-directional — it only maps outbound connections initiated from within the VPC.

> **Cost note:** NAT Gateways incur hourly charges (~$0.045/hr) plus per-GB data processing fees. For environments where cost is a concern, VPC Endpoints for AWS services (S3, SSM) can significantly reduce NAT Gateway data transfer costs.

---

## Route Tables

Two route tables control traffic flow within the VPC.

### Public Route Table

**Associated subnets:** `public-subnet-1a`, `public-subnet-1b`

| Destination | Target | Purpose |
|---|---|---|
| `10.0.0.0/16` | Local | VPC-internal traffic |
| `0.0.0.0/0` | Internet Gateway | All other traffic exits to internet |

### Private Route Table

**Associated subnets:** `private-subnet-1a`

| Destination | Target | Purpose |
|---|---|---|
| `10.0.0.0/16` | Local | VPC-internal traffic |
| `0.0.0.0/0` | NAT Gateway | Outbound-only internet access |

The critical difference: the private route table routes `0.0.0.0/0` to the **NAT Gateway**, not the Internet Gateway. This is what makes the private subnet private — there is no route that would allow inbound internet traffic to reach the EC2 instance.

---

## Security Groups

Security Groups act as **stateful, instance-level firewalls**. Unlike Network ACLs (stateless), Security Groups automatically allow return traffic for established connections — only inbound rules need to be defined for response traffic.

### ALB Security Group — `alb-sg`

| Direction | Protocol | Port | Source | Purpose |
|---|---|---|---|---|
| Inbound | TCP | 443 | `0.0.0.0/0` | Accept HTTPS from internet |
| Inbound | TCP | 80 | `0.0.0.0/0` | Accept HTTP (redirect to HTTPS) |
| Outbound | TCP | 80 | EC2 Security Group | Forward traffic to EC2 |

The ALB accepts traffic from the entire internet on ports 80 and 443. AWS WAF sits upstream and filters malicious requests before they reach the ALB.

---

### EC2 Security Group — `ec2-sg`

| Direction | Protocol | Port | Source | Purpose |
|---|---|---|---|---|
| Inbound | TCP | 80 | `alb-sg` (Security Group ID) | Accept traffic from ALB only |
| Inbound | TCP | 22 | — | **Not configured — SSH disabled** |
| Outbound | TCP | 443 | `0.0.0.0/0` | SSM Agent, AWS API calls |

**Key design decision:** The EC2 inbound rule references the **ALB Security Group ID** directly — not a CIDR range. This means only traffic originating from the ALB is permitted, even if another resource within the VPC were to attempt a direct connection to the EC2 instance.

Port 22 has no inbound rule. SSH access is not possible from any network path.

---

## Security Group Chain

The relationship between security groups creates an explicit, verifiable trust chain:

```
Internet
    │  (ports 80, 443)
    ▼
[alb-sg] — ALB
    │  (port 80, source: alb-sg only)
    ▼
[ec2-sg] — EC2 Instance
```

No other traffic path exists. The EC2 instance cannot be reached from within the VPC unless the source is the ALB security group.

---

## Administrative Access

SSH access to the EC2 instance is fully disabled. Administration is performed exclusively via **AWS Systems Manager Session Manager**.

| Attribute | SSH | SSM Session Manager |
|---|---|---|
| Port 22 required | Yes | No |
| Security Group rule needed | Yes | No |
| Key pair required | Yes | No |
| Works in private subnet | Requires bastion | Yes, natively |
| Audit trail | Optional | AWS CloudTrail (automatic) |

For full details on IAM configuration for SSM access, see [`iam-configuration.md`](./iam-configuration.md).

---

## Full Network Traffic Flow

The complete path for a legitimate user request:

```
1. User → DNS lookup → Route 53 → ALB DNS name

2. User → HTTPS request → AWS WAF inspection
   └── Malicious? → HTTP 403, dropped
   └── Clean?     → forwarded to ALB

3. ALB (public-subnet-1a or 1b)
   └── TLS terminated using ACM certificate
   └── HTTP forwarded internally to EC2 target

4. EC2 (private-subnet-1a)
   └── Nginx processes request
   └── Response returned via ALB to user

5. ALB → access log entry written to Amazon S3
```

---

## Network Security Summary

| Control | Mechanism | Enforced at |
|---|---|---|
| No direct internet access to EC2 | Private subnet, no IGW route | VPC routing layer |
| Only ALB can reach EC2 | Security Group source referencing | Instance firewall layer |
| No SSH access | No port 22 inbound rule | Security Group |
| Outbound-only internet for EC2 | NAT Gateway | Routing layer |
| Application-layer threat filtering | AWS WAF | Pre-ALB |
| Encrypted client communication | ACM certificate on ALB | Transport layer |

---

## Future Improvements

### Multi-AZ Compute

Currently, only one EC2 instance is deployed in a single AZ. Deploying instances in multiple AZs behind the ALB would provide fault tolerance against AZ-level failures and enable zero-downtime deployments.

### VPC Endpoints

Traffic from the EC2 instance to AWS services (S3, SSM, CloudWatch) currently routes through the NAT Gateway. Replacing these with **VPC Interface Endpoints** or **Gateway Endpoints** would:

- Keep AWS service traffic entirely within the AWS network
- Reduce NAT Gateway data costs
- Eliminate internet exposure for internal AWS API calls

Recommended endpoints for this architecture:

| Service | Endpoint Type |
|---|---|
| Amazon S3 | Gateway Endpoint (free) |
| AWS Systems Manager | Interface Endpoint |
| Amazon CloudWatch | Interface Endpoint |

### AWS Network Firewall

For deeper packet inspection and stateful network-layer filtering, **AWS Network Firewall** can be deployed in a dedicated inspection VPC or inline within the existing VPC. This provides:

- Intrusion Detection/Prevention (IDS/IPS)
- Domain-based outbound filtering (block egress to known malicious domains)
- Protocol anomaly detection

### Flow Logs

Enable **VPC Flow Logs** to capture metadata for all accepted and rejected traffic across the VPC. Flow logs stored in S3 and queried via Athena enable network-level anomaly detection and incident investigation.

```
Captured per flow:
- Source/destination IP and port
- Protocol
- Bytes transferred
- Accept / Reject decision
- Timestamp
```