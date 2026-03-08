# Architecture Design — Secure Web Application on AWS

## Overview

This document details the architecture and security design of the **Secure Web Application on AWS** project. The system is built around a **layered, defense-in-depth model** that keeps backend compute resources fully isolated from direct internet exposure while serving web traffic reliably and securely.

### Security Principles

| Principle | Implementation |
|---|---|
| Defense in depth | Multiple independent security layers across network, application, and transport |
| Network isolation | EC2 instance confined to a private subnet with no public IP |
| Encrypted communication | TLS enforced via AWS Certificate Manager at the load balancer |
| Application-layer filtering | AWS WAF with managed rule groups blocking common attack patterns |
| Audit & monitoring | ALB access logs persisted to Amazon S3 for forensic analysis |

---

## High-Level Architecture

Incoming requests pass through several security and networking layers before reaching the backend server. No component is a single point of control — each layer independently enforces its own security boundary.

![Architecture-diagram](architecure.png)

---

## Network Architecture

The application runs inside a custom **Amazon VPC**, providing full control over network topology and traffic flow.

### VPC Layout

```
VPC
├── Public Subnet A  ──── Application Load Balancer
├── Public Subnet B  ──── NAT Gateway
└── Private Subnet   ──── EC2 Instance (Nginx)
```

### Public Subnets

Public subnets host internet-facing infrastructure only. Resources here are limited to:

- **Application Load Balancer** — receives and terminates inbound HTTPS traffic
- **NAT Gateway** — provides controlled outbound internet access for the private subnet (e.g., OS updates, package installs)

### Private Subnet

The backend EC2 instance is deployed in a **private subnet** with no public IP address. It cannot be reached directly from the internet under any circumstance. All inbound traffic must originate from the ALB, enforced by Security Group rules.

This design eliminates an entire class of attack vectors by making the origin server unreachable from the public internet.

---

## Traffic Flow

The following describes a full request lifecycle from browser to backend.

```
1. User navigates to https://websecureapp.luizcloud.com

2. Route 53 resolves the domain to the ALB's DNS name

3. AWS WAF inspects the incoming request against managed rule groups
   └── Malicious request? → HTTP 403 Forbidden (request dropped)
   └── Clean request?    → forwarded to ALB

4. ALB terminates TLS using the ACM-issued certificate
   └── HTTP requests on port 80 are redirected to HTTPS (port 443)

5. ALB forwards the decrypted request to the EC2 target in the private subnet

6. Nginx processes the request and returns the response

7. ALB logs the full request record to Amazon S3
```

---

## Security Architecture

### 1. Network Isolation

The EC2 instance has no public IP address and resides in a subnet with no route to the internet gateway. Access is restricted at two levels:

- **Subnet routing** — no inbound route from the internet
- **Security Groups** — EC2 security group only permits inbound traffic from the ALB security group

This means even if the ALB were misconfigured, direct access to the origin would still be blocked at the network layer.

---

### 2. TLS Encryption

HTTPS is enforced using a certificate issued and managed by **AWS Certificate Manager (ACM)**. TLS is terminated at the Application Load Balancer.

Traffic between the ALB and EC2 travels over HTTP within the private VPC network. This is a deliberate and accepted tradeoff — see [Tradeoffs](#tradeoffs) for discussion.

---

### 3. Web Application Firewall (AWS WAF)

AWS WAF sits in front of the ALB and filters requests based on the following managed rule groups:

| Rule Group | Protection |
|---|---|
| AWS Managed Common Rule Set | OWASP Top 10 — XSS, LFI, RFI, and more |
| SQL Injection Protection | Blocks SQLi patterns in query strings, headers, and body |
| Known Bad Inputs | Filters payloads associated with known exploitation frameworks |
| Bot Control | Detects and blocks automated, non-browser traffic |

WAF rules are evaluated before the request reaches the load balancer, ensuring malicious traffic never touches the application layer.

---

### 4. HTTP Security Headers

Nginx is configured to include the following response headers on all requests:

| Header | Protection |
|---|---|
| `Strict-Transport-Security` | Prevents protocol downgrade attacks and enforces HTTPS |
| `X-Frame-Options: DENY` | Blocks clickjacking via iframe embedding |
| `X-Content-Type-Options: nosniff` | Prevents MIME-type sniffing by browsers |
| `Referrer-Policy: no-referrer` | Stops referrer information leakage to third parties |

---

### 5. Access Logging

ALB access logs are enabled and streamed to a dedicated **Amazon S3 bucket**. Each log entry captures:

- Client IP address
- Request method and path
- HTTP response status code
- Timestamp
- User-agent string
- SSL cipher and protocol version

Logs provide a full, immutable audit trail for traffic analysis, incident investigation, and compliance purposes. Amazon Athena can be used to query logs at scale directly from S3.

---

## Design Decisions

### TLS Termination at the ALB

Terminating TLS at the load balancer simplifies certificate lifecycle management (ACM handles renewal automatically) and removes cryptographic overhead from the EC2 instance. The tradeoff — HTTP traffic within the VPC — is covered in the [Tradeoffs](#tradeoffs) section.

### Private Subnet for EC2

Placing the origin server in a private subnet is standard practice for production cloud workloads. It ensures the application is never directly reachable from the internet, regardless of application-level misconfigurations.

### AWS Managed WAF Rules

Managed rule groups were chosen over custom rules to ensure broad, maintained coverage against evolving threat patterns without requiring ongoing rule authorship. AWS updates managed rules as new threats emerge.

---

## Tradeoffs

### HTTP Between ALB and EC2

**Concern:** TLS terminates at the ALB; traffic to the EC2 instance is unencrypted HTTP.

**Justification:** Communication travels entirely within a private subnet over AWS's internal network. There is no route for this traffic to leave the VPC boundary. For most architectures this is an acceptable tradeoff.

**Alternative:** End-to-end TLS can be implemented by installing a certificate on the EC2 instance and configuring the ALB target group to use HTTPS. This is recommended for environments with strict compliance requirements (e.g., PCI-DSS, HIPAA).

---

### NAT Gateway Cost

**Concern:** NAT Gateways incur hourly charges plus per-GB data transfer fees.

**Justification:** The NAT Gateway enables the private EC2 instance to reach the internet for outbound traffic (package updates, AWS API calls) without exposing it inbound. The security benefit justifies the cost for any production workload.

**Alternative:** VPC Endpoints can reduce NAT Gateway data costs for traffic destined for AWS services (e.g., S3, SSM).

---

### Bot Control May Block Legitimate Clients

**Concern:** The WAF Bot Control rule group may block command-line tools like `curl` or `wget`.

**Justification:** This is expected behavior. Automated non-browser clients are treated as potential threats by default.

**Alternative:** Allowlist specific IP ranges or user-agents in WAF rules for known legitimate automated clients.

---

## Future Improvements

### Amazon CloudFront

Placing CloudFront in front of the ALB would add:

- **Edge caching** — reduced latency for static assets
- **DDoS mitigation** — AWS Shield Standard included by default
- **Geo-restriction** — block traffic from specific regions
- **WAF at the edge** — shift filtering even further from the origin

### Infrastructure as Code

The current architecture was provisioned manually. Migrating to IaC would enable:

- Reproducible, version-controlled deployments
- Environment parity between staging and production
- Automated rollback on failure

Suitable tools: **Terraform**, **AWS CloudFormation**

### Enhanced Security Monitoring

| Service | Capability |
|---|---|
| AWS GuardDuty | Threat detection using ML on VPC flow logs and DNS logs |
| AWS Security Hub | Centralized security posture and compliance dashboard |
| AWS CloudTrail | Full API audit trail across all AWS accounts and regions |

### Log Analysis with Amazon Athena

ALB logs stored in S3 can be queried directly using Athena — enabling SQL-based traffic analysis, anomaly detection, and custom dashboards without moving data out of S3.

---


