# WAF Configuration — AWS Web Application Firewall

## Overview

This document describes the AWS WAF configuration used to protect the web application at the application layer. The WAF is attached directly to the **Application Load Balancer (ALB)** and inspects every inbound HTTP/HTTPS request before it is forwarded to the backend EC2 instance.

The firewall operates at **Layer 7 (application layer)**, enabling it to inspect request content — URLs, headers, query strings, and body payloads — rather than just IP addresses and ports.

---

## Deployment Position

```
User Request
      │
      ▼
Amazon Route 53          ← DNS resolution
      │
      ▼
AWS WAF  ◄─────────────── All requests inspected here
      │
      ▼
Application Load Balancer (HTTPS)
      │
      ▼
EC2 Instance — Private Subnet (Nginx)
```

The WAF sits upstream of the ALB, meaning **no malicious request ever reaches the load balancer or origin server**. Blocked requests are rejected at the WAF layer with an `HTTP 403 Forbidden` response.

---

## Web ACL

A **Web ACL (Access Control List)** defines the ordered set of rules applied to incoming traffic.

| Field | Value |
|---|---|
| **Web ACL Name** | `secure-web-waf` |
| **Scope** | Regional |
| **Associated Resource** | Application Load Balancer |
| **Default Action** | `Allow` (requests not matching any rule) |
| **Rule Evaluation** | Rules evaluated in priority order; first match wins |

The default action is `Allow` because the managed rule groups below are responsible for explicitly blocking known-bad traffic. Any request that passes all rules is considered clean and forwarded to the ALB.

---

## Managed Rule Groups

All rule groups used in this configuration are **AWS Managed Rules** — maintained and updated by the AWS Threat Intelligence team. This means protections are automatically updated as new attack patterns emerge, without requiring manual rule authorship.

The four rule groups are evaluated in the following priority order:

---

### 1. Core Rule Set — `AWSManagedRulesCommonRuleSet`

**Priority:** 1
**Action on match:** Block (`HTTP 403`)

The Core Rule Set provides broad protection against the OWASP Top 10 and other common web application vulnerabilities.

| Rule | Protection |
|---|---|
| `CrossSiteScripting_BODY` | XSS payloads in request body |
| `CrossSiteScripting_QUERYARGUMENTS` | XSS payloads in query strings |
| `GenericRFI_BODY` | Remote file inclusion attempts |
| `GenericLFI_QUERYARGUMENTS` | Local file inclusion via path traversal |
| `RestrictedExtensions_URIPATH` | Requests for sensitive file extensions (`.env`, `.bak`, etc.) |
| `EC2MetaDataSSRF_BODY` | SSRF attempts targeting the EC2 metadata service |
| `UserAgent_BadBots_HEADER` | Known bad user-agent strings |

This rule group alone covers a significant portion of common web attack traffic.

---

### 2. SQL Injection Protection — `AWSManagedRulesSQLiRuleSet`

**Priority:** 2
**Action on match:** Block (`HTTP 403`)

Detects and blocks SQL injection patterns across all inspectable request components — query strings, URI paths, headers, and body.

**Verified test result:**

```
Request:
GET https://websecureapp.luizcloud.com/?id=1' OR '1'='1

Response:
HTTP 403 Forbidden
```

The request was blocked at the WAF layer. The backend server received no traffic.

SQLi rule coverage includes:

- Classic `OR 1=1` pattern matching
- UNION-based injection
- Blind SQLi timing patterns
- Stacked queries

---

### 3. Known Bad Inputs — `AWSManagedRulesKnownBadInputsRuleSet`

**Priority:** 3
**Action on match:** Block (`HTTP 403`)

Filters requests containing payload patterns associated with known exploitation frameworks and attack tools.

| Pattern Type | Example |
|---|---|
| Log4j / Log4Shell | `${jndi:ldap://...}` in headers or body |
| JavaDeserializationExploits | Known Java deserialization gadget chains |
| PROPFIND method abuse | WebDAV-based enumeration |
| Command injection signatures | Shell metacharacters in user-supplied fields |

This rule group is particularly valuable for catching exploit attempts targeting infrastructure-level vulnerabilities (e.g., Log4Shell), not just application-level logic flaws.

---

### 4. Bot Control — `AWSManagedRulesBotControlRuleSet`

**Priority:** 4
**Action on match:** Block (`HTTP 403`)

Identifies and blocks non-human automated traffic using browser fingerprinting, behavioral signals, and known bot signatures.

| Bot Category | Examples |
|---|---|
| Scrapers | Content harvesting bots |
| Scanners | Vulnerability scanners (Nikto, Nessus) |
| Command-line clients | `curl`, `wget`, `python-requests` |
| Known bot signatures | Bots with registered user-agent signatures |

**Verified test result:**

```
Request:
curl https://websecureapp.luizcloud.com

Response:
HTTP 403 Forbidden
```

The Bot Control rule group correctly identified the request as automated traffic and blocked it before it reached the ALB.

> **Note:** This behavior is intentional and expected. If legitimate automated clients (e.g., health check scripts, CI/CD pipelines) need access, their IP ranges or user-agents can be allowlisted via a custom rule with higher priority than the Bot Control group.

---

## Rule Evaluation Order

WAF rules are evaluated top-down by priority. The first matching rule determines the outcome — no further rules are evaluated for that request.

```
Priority 1 → Core Rule Set           (OWASP, XSS, LFI/RFI, SSRF)
Priority 2 → SQL Injection Rule Set  (SQLi across all request components)
Priority 3 → Known Bad Inputs        (exploit frameworks, Log4Shell)
Priority 4 → Bot Control             (automated and non-browser traffic)
      │
      ▼
No match → Default Action: Allow → Forward to ALB
```

---

## Security Testing Results

| Test | Payload | WAF Rule Triggered | Response |
|---|---|---|---|
| SQL Injection | `?id=1' OR '1'='1` | `AWSManagedRulesSQLiRuleSet` | `HTTP 403` |
| Bot detection | `curl` request | `AWSManagedRulesBotControlRuleSet` | `HTTP 403` |
| XSS attempt | `?q=<script>alert(1)</script>` | `AWSManagedRulesCommonRuleSet` | `HTTP 403` |
| Clean browser request | Normal GET | No rule matched | `HTTP 200` |

All malicious test requests were blocked at the WAF layer. No traffic reached the ALB or EC2 instance.

---

## Monitoring & Visibility

WAF metrics are published to **Amazon CloudWatch** in near real-time and are also visible in the **AWS WAF console dashboard**.

| Metric | Description |
|---|---|
| `AllowedRequests` | Count of requests passed to the ALB |
| `BlockedRequests` | Count of requests rejected by WAF rules |
| `CountedRequests` | Requests matched by count-only rules (for analysis) |
| `PassedRequests` | Requests that matched no rule and hit the default action |

Rule-level metrics are available per rule group, enabling drill-down into which rule group is triggering most frequently.

For deeper analysis, WAF logs can be streamed to:

- **Amazon S3** — long-term retention and querying with Athena
- **Amazon CloudWatch Logs** — real-time alerting and dashboards
- **Amazon Kinesis Data Firehose** — streaming to SIEM tools

---

## Current Limitations & Tradeoffs

**Bot Control blocks legitimate CLI traffic**

The Bot Control rule group is broad by design. Any request that doesn't present a recognized browser fingerprint is a candidate for blocking. This is the right default for a public-facing application but requires allowlisting for known automated clients.

**No rate limiting configured**

The current configuration does not include rate-based rules. A sustained high-volume request from a single IP (e.g., credential stuffing or brute force) would not be blocked unless it triggered a content-based rule. Rate limiting is flagged as a future improvement.

**WAF does not protect against volumetric DDoS**

AWS WAF operates at Layer 7 and is not designed to absorb volumetric network-layer attacks. For DDoS protection, **AWS Shield Advanced** or **Amazon CloudFront with Shield** should be layered in front.

---

## Future Improvements

### Rate-Based Rules

Add rules to block IPs that exceed a configurable request threshold within a rolling time window. This directly mitigates brute-force login attempts, credential stuffing, and scraping at scale.

```
Rule Type: Rate-based
Threshold: 1000 requests / 5 minutes per IP
Action: Block (temporary, auto-expires)
```

### Custom Application Rules

Application-specific rules can be written to restrict access patterns that managed rules don't cover — for example:

- Blocking requests to admin paths (`/admin`, `/wp-admin`) from non-whitelisted IPs
- Rejecting requests with unexpected `Content-Type` headers
- Enforcing expected request size limits

### AWS Shield Advanced Integration

Pairing WAF with **AWS Shield Advanced** adds:

- Layer 3/4 DDoS volumetric protection
- 24/7 access to the AWS DDoS Response Team (DRT)
- Cost protection during DDoS events

### Athena Log Analysis

Stream WAF logs to S3 and query with **Amazon Athena** to build custom dashboards showing blocked request trends, top attacking IPs, and most-triggered rule groups over time.