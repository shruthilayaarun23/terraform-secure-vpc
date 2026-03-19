# Terraform-secure-vpc

A production-inspired AWS VPC built with security embedded at authoring time,
not patched in after deployment. Infrastructure is defined in Terraform,
scanned with Checkov on every pull request, and state is stored remotely with
encryption and locking.

---

## Security metrics

| Metric                                            | Value            |
|---------------------------------------------------|------------------|
| Checkov checks passing                            | 14               |
| Checkov checks skipped (documented)               | 6                |
| Checkov checks failing                            | 0                |
| Hardcoded credentials                             | 0                |
| Security groups with open SSH (port 22)           | 0                |
| Security groups with open RDP (port 3389)         | 0                |
| Subnets with public IP auto-assign (private tier) | 0                |
| State file encryption                             | AES-256 (S3 SSE) |
| Log retention                                     | 365 days         |
| IAM roles with AdministratorAccess                | 0                |

---

## Architecture

```
                          Internet
                             │
                    ┌────────▼────────┐
                    │ Internet Gateway │
                    └────────┬────────┘
                             │
          ┌──────────────────▼──────────────────┐
          │           VPC  10.0.0.0/16          │
          │                                     │
          │  ┌───────────────────────────────┐  │
          │  │   Public Subnet 10.0.1.0/24   │  │
          │  │                               │  │
          │  │  [ Load Balancer / NAT GW ]   │  │
          │  │   Public IPs assigned         │  │
          │  └──────────────┬────────────────┘  │
          │                 │ internal only     │
          │  ┌──────────────▼────────────────┐  │
          │  │   Private Subnet 10.0.2.0/24  │  │
          │  │                               │  │
          │  │  [ App Servers / Databases ]  │  │
          │  │   No public IPs               │  │
          │  └───────────────────────────────┘  │
          │                                     │
          │  Default SG: all traffic blocked    │
          │  Flow logs: ALL traffic → CloudWatch│
          └─────────────────────────────────────┘

State backend
  └── S3 bucket (AES-256 encrypted, public access blocked)
        └── DynamoDB lock table (prevents concurrent applies)

CI pipeline (GitHub Actions)
  └── Checkov scan runs on every pull request
        └── Blocks merge if new violations introduced
```

---

## Resources provisioned

| Resource                    | Purpose                                       |
|-----------------------------|-----------------------------------------------|
| `aws_vpc`                   | Main network — `10.0.0.0/16`                  |
| `aws_subnet` (public)       | Load balancer tier — `10.0.1.0/24`            |
| `aws_subnet` (private)      | Application tier — `10.0.2.0/24`              |
| `aws_internet_gateway`      | Outbound internet access for public subnet    |
| `aws_security_group`        | Web traffic — HTTPS/HTTP in, all out, no SSH  |
| `aws_default_security_group`| Locked down — no ingress or egress            |
| `aws_flow_log`              | Captures all VPC traffic metadata             |
| `aws_cloudwatch_log_group`  | Stores flow logs for 365 days                 |
| `aws_iam_role`              | Least-privilege role for flow log service only|

---

## Security controls

### Network isolation
Traffic from the internet can only enter through the public subnet. Private
subnet resources have no public IPs and cannot be reached directly from
outside the VPC. This limits blast radius — a compromised load balancer
cannot directly reach the database tier.

### No open administrative ports
The web security group has no rules for SSH (22) or RDP (3389). In a
production environment, administrative access would use AWS Systems Manager
Session Manager — no open ports, full session audit trail in CloudTrail.

### Default security group locked
AWS creates a default security group per VPC that allows all traffic between
members. This project overrides it explicitly with zero rules — nothing can
accidentally be assigned a permissive default.

### Encrypted remote state
Terraform state is stored in S3 with AES-256 encryption and all public
access blocked. A DynamoDB table provides distributed locking to prevent
state corruption from concurrent operations.

### VPC flow logging
All accepted and rejected traffic is logged to CloudWatch with a 365-day
retention period. Flow logs are the primary forensic tool during a security
incident — they record source IP, destination IP, port, protocol, and
whether the connection was allowed or denied.

### IAM least privilege
The IAM role for VPC flow logging is scoped to a single AWS service
principal (`vpc-flow-logs.amazonaws.com`). No managed policies. No wildcard
permissions. No cross-account trust.

---

## Checkov scan summary

All 14 passing checks are enforced controls. All 6 skipped checks have
documented rationale embedded directly in the Terraform code as
`checkov:skip` comments — the decision is auditable without external docs.

```
Passed checks: 14, Failed checks: 0, Skipped checks: 6
```

| Check ID           | Status | Control                                                    |
|--------------------|--------|------------------------------------------------------------|
| CKV_AWS_41         | PASS   | No hardcoded credentials                                   |
| CKV_AWS_24         | PASS   | No SSH open to internet                                    |
| CKV_AWS_25         | PASS   | No RDP open to internet                                    |
| CKV_AWS_277        | PASS   | No all-ports ingress rule                                  |
| CKV_AWS_23         | PASS   | All SG rules have descriptions                             |
| CKV2_AWS_11        | PASS   | VPC flow logging enabled                                   |
| CKV2_AWS_12        | PASS   | Default SG restricts all traffic                           |
| CKV_AWS_66         | PASS   | Log group retention set                                    |
| CKV_AWS_338        | PASS   | Log retention ≥ 1 year                                     |
| CKV_AWS_60         | PASS   | IAM role scoped to specific service                        |
| CKV_AWS_61         | PASS   | No assume-role wildcard                                    |
| CKV_AWS_274        | PASS   | No AdministratorAccess policy                              |
| CKV2_AWS_56        | PASS   | No IAMFullAccess policy                                    |
| CKV_AWS_130        | PASS   | Private subnet blocks public IPs                           |
| CKV_AWS_130        | SKIP   | Public subnet assigns IPs — intentional for LB tier        |
| CKV_AWS_260        | SKIP   | Port 80 open — required for HTTP→HTTPS redirect            |
| CKV_AWS_382        | SKIP   | Unrestricted egress — web servers need external API access |
| CKV_AWS_158        | SKIP   | KMS not used — dev environment, cost not justified         |
| CKV2_AWS_5         | SKIP   | SG not yet attached — EC2 instance not in scope            |

---

## CI/CD pipeline

A GitHub Actions workflow runs Checkov on every pull request targeting `main`.
The pipeline fails if any new violation is introduced — infrastructure with
security issues cannot be merged.

```
Pull request opened
       │
       ▼
GitHub Actions triggers
       │
       ▼
Checkov scans all .tf files
       │
    ┌──┴──┐
  PASS   FAIL
    │      │
    ▼      ▼
Merge   PR blocked
allowed
---

## VPC flow logs — live threat data

Within minutes of deploying a test instance into the public subnet, VPC flow
logs captured unsolicited connection attempts from external IPs across the
internet. Every single one was rejected by the security group.

Zero accepted connections from the internet. The security controls worked.

```
SOURCE IP         DEST PORT   PROTOCOL   ACTION   NOTES
198.235.24.12     1244        TCP        REJECT   uncommon port — botnet scanner
89.248.163.200    9999        TCP        REJECT   known malicious IP, backdoor probe
64.89.161.43      80          TCP        REJECT   web scanner
66.132.153.159    636         TCP        REJECT   LDAP over SSL probe
107.150.97.138    8381        TCP        REJECT   port scan
185.156.73.182    8446        TCP        REJECT   known scanner IP
13.219.1.233      162         TCP        REJECT   SNMP probe
73.30.107.113     0           ICMP       REJECT   ping sweep
85.217.149.58     14406       TCP        REJECT   coordinated port scan
85.217.149.56     15722       TCP        REJECT   coordinated port scan (same /24 subnet)
```

The last two entries — `85.217.149.58` and `85.217.149.56` — are from the
same `/24` subnet hitting different high ports within the same capture window.
That is a coordinated port scan from two IPs working in tandem.


## What I would add in production

- **KMS customer-managed keys** for flow log encryption
- **AWS Config rules** for continuous drift detection
- **Terraform Sentinel policies** for org-wide guardrails
- **VPC endpoints** for S3 and DynamoDB (no state traffic over internet)
- **Multi-account architecture** — separate networking, workload, and security accounts
- **NAT Gateway** for private subnet outbound internet access
- **Flow logs → SIEM** (Splunk / AWS Security Hub) for correlation

---
