# Security Group Configuration

## Overview

This document details the Security Group rules applied to each component in the architecture. Security Groups act as **stateful, instance-level firewalls** — return traffic for established connections is automatically permitted, so only inbound rules require explicit definition.

Two security groups are used in this architecture, forming a **chained trust model** where the EC2 instance only accepts traffic that has already passed through the ALB.

---

## Security Group Architecture

```
Internet
    │  HTTPS (443) / HTTP (80)
    ▼
┌─────────────────────────────┐
│   ALB Security Group        │  ← accepts public web traffic
│   alb-sg                    │
└─────────────────────────────┘
    │  HTTP (80) — source: alb-sg only
    ▼
┌─────────────────────────────┐
│   EC2 Security Group        │  ← accepts traffic from ALB only
│   ec2-sg                    │
└─────────────────────────────┘
```

No other traffic path exists. The EC2 instance cannot be reached from the internet, from other VPC resources, or via SSH under any circumstance.

---

## ALB Security Group — `alb-sg`

The ALB is the only component with direct internet exposure. Its security group permits inbound web traffic on both HTTP and HTTPS.

### Inbound Rules

| Rule | Protocol | Port | Source | Reason |
|---|---|---|---|---|
| Allow HTTPS | TCP | 443 | `0.0.0.0/0` | Accept encrypted web traffic from all clients |
| Allow HTTP | TCP | 80 | `0.0.0.0/0` | Accept HTTP for redirect to HTTPS |

> **Why allow port 80?** HTTP traffic is accepted solely to redirect clients to HTTPS via an ALB listener rule (`HTTP 301 → HTTPS`). No unencrypted content is served — the redirect enforces secure communication for clients that navigate to the `http://` URL directly.

### Outbound Rules

| Rule | Protocol | Port | Destination | Reason |
|---|---|---|---|---|
| Allow all | All | All | `0.0.0.0/0` | Permit response traffic and ALB health checks to EC2 |

The broad outbound rule is standard for ALBs. Actual traffic forwarding is constrained by the target group configuration — the ALB only forwards to registered EC2 targets on port 80.

---

## EC2 Security Group — `ec2-sg`

The EC2 instance security group is locked down to accept **only traffic originating from the ALB**. This is enforced by referencing the ALB Security Group ID as the source — not a CIDR block.

### Inbound Rules

| Rule | Protocol | Port | Source | Reason |
|---|---|---|---|---|
| Allow HTTP | TCP | 80 | `alb-sg` (Security Group ID) | Accept forwarded requests from ALB only |

### Explicitly Blocked

| Rule | Protocol | Port | Source | Reason |
|---|---|---|---|---|
| ❌ SSH | TCP | 22 | Any | No SSH access — managed via AWS SSM Session Manager |

Port 22 has no inbound rule. SSH is not accessible from any network path, including within the VPC.

### Outbound Rules

| Rule | Protocol | Port | Destination | Reason |
|---|---|---|---|---|
| Allow all | All | All | `0.0.0.0/0` | SSM Agent communication, AWS API calls, OS updates via NAT |

---

## Why Source Referencing Matters

The EC2 inbound rule uses the **ALB Security Group ID** as its source rather than a CIDR range. This is a critical distinction.

| Approach | Rule Source | Risk |
|---|---|---|
| CIDR-based | `10.0.0.0/16` | Any resource in the entire VPC can reach EC2 on port 80 |
| Security Group reference | `alb-sg` | **Only** the ALB can reach EC2 on port 80 — nothing else in the VPC |

Using a Security Group reference creates a precise, identity-based trust boundary. Even if another EC2 instance or service were launched inside the VPC, it would not be able to reach the web server directly.

---

## Port 22 — SSH Disabled by Design

SSH access is not configured and is intentionally absent from the EC2 security group. Administrative access to the instance is handled exclusively through **AWS Systems Manager Session Manager**, which requires:

- No open inbound port
- No key pair
- No bastion host

This eliminates the most common attack vector against EC2 instances — exposed SSH ports targeted by automated brute-force and credential-stuffing tools.

For full details, see [`iam-configuration.md`](./iam-configuration.md).

---

## Security Group Summary

| Security Group | Attached To | Key Inbound Rule | SSH | Exposed to Internet |
|---|---|---|---|---|
| `alb-sg` | Application Load Balancer | 443, 80 from `0.0.0.0/0` | No | Yes (intentional) |
| `ec2-sg` | EC2 Instance (Nginx) | Port 80 from `alb-sg` only | No | No |

---

## Future Improvements

### Restrict ALB Outbound

The current ALB outbound rule allows all traffic. This can be tightened to explicitly permit only port 80 to the EC2 security group, reducing the rule surface further.

### VPC Endpoint Security Groups

If VPC Interface Endpoints are added for services like SSM or CloudWatch, dedicated security groups should be created for each endpoint — allowing inbound 443 only from `ec2-sg`. This keeps all AWS service traffic off the NAT Gateway entirely.

### EC2 Egress Restriction

The EC2 outbound rule currently allows all traffic. For a hardened configuration, outbound rules can be restricted to:

| Protocol | Port | Destination | Purpose |
|---|---|---|---|
| TCP | 443 | `0.0.0.0/0` | SSM Agent, AWS APIs, HTTPS |
| TCP | 80 | — | Removed (no HTTP egress needed) |

This ensures the instance cannot initiate unexpected outbound connections even if compromised.