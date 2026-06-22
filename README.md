# 🏛️ lagos-lawfirm-s3-dms
### AWS S3 Document Management System — Lagos Law Firm

> A secure, scalable cloud document management solution built on Amazon S3 for a Lagos-based law firm. Supports encrypted storage, lifecycle-managed archiving, and pre-signed URL access for confidential legal documents.

---

## 📋 Table of Contents

- [Architecture Overview](#architecture-overview)
- [Security Controls](#security-controls)
- [Lifecycle Policy](#lifecycle-policy)
- [Pre-Signed URL Workflow](#pre-signed-url-workflow)
- [Cost Estimate](#cost-estimate)
- [Screenshots](#screenshots)
- [Getting Started](#getting-started)

---

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
│          Law Firm Staff  /  Partner Portal  /  Clients          │
└──────────────────────────┬──────────────────────────────────────┘
                           │  HTTPS (Pre-Signed URLs)
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                      AWS SERVICES LAYER                         │
│                                                                 │
│   ┌────────────┐    ┌─────────────┐    ┌────────────────────┐  │
│   │  Amazon    │    │   AWS IAM   │    │   AWS CloudTrail   │  │
│   │  S3 Bucket │◄───│   Policies  │    │   (Audit Logs)     │  │
│   │            │    │  & Roles    │    │                    │  │
│   └─────┬──────┘    └─────────────┘    └────────────────────┘  │
│         │                                                      │
│   ┌─────▼──────┐    ┌─────────────┐    ┌────────────────────┐  │
│   │  S3 Server │    │  Amazon     │    │   Amazon SNS /     │  │
│   │  -Side Enc │    │  KMS Keys   │    │   EventBridge      │  │
│   │  (SSE-KMS) │◄───│  (CMK)      │    │   (Notifications)  │  │
│   └────────────┘    └─────────────┘    └────────────────────┘  │
│                                                                │
│   ┌──────────────────────────────────────────────────────────┐ │
│   │              S3 BUCKET STRUCTURE                         │ │
│   │                                                          │ │
│   │  lagos-lawfirm-docs-simeonjnr/                           │ │
│   │  ├── contracts/       (Signed agreements)                │ │
│   │  ├── contracts/        (Signed agreements)               │ │
│   │  ├── correspondence/   (Client communications)           │ │
│   │  ├── filings/          (Court filings & exhibits)        │ │
│   │  └── archive/          (Closed matters — Glacier)        │ │
│   └──────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    LIFECYCLE MANAGEMENT                         │
│  Standard → Standard-IA (90 days) → Glacier (365 days)          │
└─────────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Service | Purpose |
|-----------|---------|---------|
| Document Store | Amazon S3 | Primary storage for all legal documents |
| Encryption | AWS KMS (CMK) | Customer-managed encryption keys |
| Access Control | IAM Roles & Policies | Least-privilege document access |
| Audit Trail | AWS CloudTrail | Full access & change logging |
| Archival | S3 Glacier | Long-term storage for closed cases |
| Notifications | Amazon SNS | Upload & access event alerts |

---

## 🔒 Security Controls

### 1. Encryption at Rest
- **Server-Side Encryption** using AWS KMS Customer Managed Keys (SSE-KMS)
- All objects encrypted by default; unencrypted uploads are blocked via bucket policy
- KMS key rotation enabled (annual automatic rotation)

### 2. Encryption in Transit
- Bucket policy enforces `aws:SecureTransport: true` — HTTP requests denied
- All data accessed exclusively over TLS 1.2+

### 3. Access Control
- **Block Public Access**: All four block public access settings enabled
- **IAM Least Privilege**: Separate roles for admin, paralegal, and read-only access
- **Bucket Policy**: Explicit deny for any principal outside the firm's AWS account
- **MFA Delete**: Enabled on bucket versioning to prevent accidental/malicious deletion

### 4. Versioning
- S3 Versioning enabled on the bucket
- Protects against overwrites and enables point-in-time recovery
- Non-current versions retained for 180 days before expiry

### 5. Audit & Monitoring
- **AWS CloudTrail**: Logs all S3 API calls (GetObject, PutObject, DeleteObject)
- **S3 Server Access Logging**: Detailed access logs stored in a separate logging bucket
- **Amazon Macie**: Enabled for automatic sensitive data discovery (PII, legal identifiers)
- **CloudWatch Alarms**: Triggered on unusual access patterns or large data transfers

### 6. Network Controls
- S3 VPC Endpoint configured — internal access does not traverse public internet
- Bucket policy restricts access to firm's VPC and specific IAM principals

### Sample Bucket Policy (Deny HTTP)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonSSLRequests",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::lagos-lawfirm-docs-simeonjnr",
        "arn:aws:s3:::lagos-lawfirm-docs-simeonjnr/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

---

## 🔄 Lifecycle Policy

Documents follow a tiered storage lifecycle based on access frequency and retention requirements.

```
Day 0          Day 90              Day 365             Day 2555 (7 yrs)
   │               │                   │                      │
   ▼               ▼                   ▼                      ▼
S3 Standard  →  S3 Standard-IA  →  S3 Glacier        →   Permanent Delete
(Active)        (Infrequent)       (Archive / Closed)     (Per policy)
$0.023/GB       $0.0125/GB          $0.004/GB
```

### Lifecycle Rules

| Rule Name | Prefix | Transition | Action |
|-----------|--------|------------|--------|
| `active-to-ia` | `cases/`, `contracts/` | 90 days | Move to S3 Standard-IA |
| `ia-to-glacier` | `cases/`, `contracts/` | 365 days | Move to S3 Glacier |
| `correspondence-archive` | `correspondence/` | 180 days | Move to S3 Standard-IA |
| `evidence-retain` | `evidence/` | 730 days | Move to S3 Glacier |
| `expire-noncurrent` | All prefixes | 180 days | Delete non-current versions |
| `7yr-legal-retention` | `archive/` | 2555 days | Permanently delete (legal hold expiry) |

### Sample Lifecycle Configuration (JSON)
```json
{
  "Rules": [
    {
      "ID": "active-to-ia",
      "Status": "Enabled",
      "Filter": { "Prefix": "cases/" },
      "Transitions": [
        { "Days": 90, "StorageClass": "STANDARD_IA" },
        { "Days": 365, "StorageClass": "GLACIER" }
      ],
      "NoncurrentVersionExpiration": { "NoncurrentDays": 180 }
    }
  ]
}
```

---

## 🔗 Pre-Signed URL Workflow

Pre-signed URLs allow authorized users and external clients to securely upload or download documents **without requiring AWS credentials**.

### Upload Workflow (PUT)

```
┌─────────────┐        ┌──────────────────┐       ┌──────────────┐
│  Law Firm   │        │  Application /   │       │   Amazon S3  │
│  Staff/API  │        │  Lambda Function │       │   Bucket     │
└──────┬──────┘        └────────┬─────────┘       └──────┬───────┘
       │                        │                         │
       │  1. Request upload URL │                         │
       │───────────────────────►│                         │
       │                        │                         │
       │                        │ 2. GeneratePresignedUrl │
       │                        │  (PUT, 15-min expiry)   │
       │                        │◄────────────────────────│
       │                        │                         │
       │  3. Return pre-signed  │                         │
       │     URL to client      │                         │
       │◄───────────────────────│                         │
       │                        │                         │
       │  4. PUT document directly to S3 using URL        │
       │─────────────────────────────────────────────────►│
       │                        │                         │
       │  5. 200 OK             │                         │
       │◄─────────────────────────────────────────────────│
```

### Download Workflow (GET)

```
┌─────────────┐        ┌──────────────────┐       ┌──────────────┐
│  Client /   │        │  Application /   │       │   Amazon S3  │
│  Partner    │        │  Lambda Function │       │   Bucket     │
└──────┬──────┘        └────────┬─────────┘       └──────┬───────┘
       │                        │                         │
       │  1. Request document   │                         │
       │───────────────────────►│                         │
       │                        │ 2. Verify IAM auth      │
       │                        │    GeneratePresignedUrl │
       │                        │    (GET, 5-min expiry)  │
       │                        │◄────────────────────────│
       │                        │                         │
       │  3. Return pre-signed  │                         │
       │     URL (time-limited) │                         │
       │◄───────────────────────│                         │
       │                        │                         │
       │  4. GET document via pre-signed URL              │
       │─────────────────────────────────────────────────►│
       │                        │                         │
       │  5. Serve encrypted document over HTTPS          │
       │◄─────────────────────────────────────────────────│
```

### URL Expiry Policy

| Use Case | Expiry | Rationale |
|----------|--------|-----------|
| Internal staff download | 60 minutes | Active working session |
| External client download | 15 minutes | Minimize exposure window |
| Document upload (staff) | 30 minutes | Allow for large file uploads |
| Evidence submission | 10 minutes | High-sensitivity, short window |

### Sample Code (Python — boto3)
```python
import boto3

s3_client = boto3.client('s3', region_name='af-south-1')

def generate_presigned_url(bucket, key, expiry=900, operation='get_object'):
    url = s3_client.generate_presigned_url(
        ClientMethod=operation,
        Params={'Bucket': bucket, 'Key': key},
        ExpiresIn=expiry
    )
    return url

# Example: Generate download URL for a contract
url = generate_presigned_url(
    bucket='lagos-lawfirm-docs-simeonjnr',
    key='contracts/2024/client-agreement-001.pdf',
    expiry=300  # 5 minutes
)
```

---

## 💰 Cost Estimate

> Based on estimated usage for a mid-sized Lagos law firm (~20 staff, ~500 active matters/year).
> Region: **af-south-1 (Cape Town)** — closest AWS region to Lagos.

### Monthly Storage Estimate

| Storage Tier | Estimated Volume | Unit Price | Monthly Cost |
|-------------|-----------------|------------|-------------|
| S3 Standard (active docs) | 50 GB | $0.023/GB | $1.15 |
| S3 Standard-IA (90-day+ docs) | 200 GB | $0.0125/GB | $2.50 |
| S3 Glacier (archived matters) | 1 TB | $0.004/GB | $4.10 |
| S3 Logging Bucket | 5 GB | $0.023/GB | $0.12 |
| **Storage Subtotal** | | | **~$7.87/month** |

### Monthly Request & Transfer Estimate

| Service | Estimated Usage | Unit Price | Monthly Cost |
|---------|----------------|------------|-------------|
| PUT/COPY/POST requests | 10,000 | $0.0054/1K | $0.054 |
| GET requests | 50,000 | $0.00044/1K | $0.022 |
| Data Transfer OUT | 20 GB | $0.09/GB | $1.80 |
| AWS KMS API calls | 5,000 | $0.03/10K | $0.015 |
| CloudTrail (management events) | Included in free tier | — | $0.00 |
| **Requests Subtotal** | | | **~$1.89/month** |

### Total Estimated Monthly Cost

| Category | Cost |
|----------|------|
| Storage | $7.87 |
| Requests & Transfer | $1.89 |
| Amazon Macie (500 GB scanned) | $0.50 |
| **Total Estimate** | **~$10.26 / month** |

> ⚠️ Costs will scale with data volume and request rates. Use [AWS Pricing Calculator](https://calculator.aws) for precise estimates. First 5 GB of data transfer out is free under the AWS Free Tier.

---

## 📸 Screenshots

> All screenshots taken from the live AWS Console for bucket `lagos-lawfirm-docs-simeonjnr-simeonjnr` (us-east-1), created May 31, 2026.

---

### 1. S3 Bucket — Objects Organised in 3 Prefix Folders
Bucket contains three prefix folders: `contracts/`, `correspondence/`, and `filings/` — each acting as a logical document category with versioning visible via the "Show versions" toggle.

![S3 Bucket Objects — 3 prefix folders](./screenshots/s3-bucket-objects.png)

---

### 2. Bucket Properties — Versioning & ABAC
**Bucket Versioning: Enabled.** MFA Delete is currently disabled (can be enabled via CLI/SDK). Bucket ABAC (Attribute-Based Access Control) is also configured, allowing tag-based IAM policies for fine-grained access.

![Bucket Properties — Versioning enabled](./screenshots/bucket-properties-versioning.png)

---

### 3. Bucket Properties — Tags & Default Encryption
Three user-defined tags are set: `Environment: Production`, `Owner: YourName`, `Project: LawFirmDMS`. Default encryption uses **SSE-S3** with Bucket Key enabled to reduce KMS API costs.

![Bucket Tags and Encryption](./screenshots/bucket-tags-encryption.png)

---

### 4. Default Encryption Detail
Confirms **SSE-S3** as the encryption type, Bucket Key **Enabled**, and **SSE-C (Customer-Provided Keys) blocked** — preventing unmanaged external key usage.

![Default Encryption Detail](./screenshots/bucket-encryption-detail.png)

---

### 5. Pre-Signed URL — Access Denied After Expiry
After the 60-second expiry window, S3 returns `AccessDenied` with `Request has expired`. The response shows the exact expiry timestamp vs server time, proving the time-limited security model works correctly.

![Pre-Signed URL Expired](./screenshots/presigned-url-expired.png)

---

### 6. Pre-Signed URL — File Successfully Downloads in Browser
Within the validity window, the pre-signed URL serves `contracts/sample-contract.pdf` directly in the browser (showing "Version 1 — original data") and triggers a download prompt — demonstrating successful authenticated access without AWS credentials.

![Pre-Signed URL Working](./screenshots/presigned-url-working.png)

---

### 7. Lifecycle Rule — `LawFirmArchive` (Enabled)
Lifecycle configuration page showing the `LawFirmArchive` rule is **Enabled** and scoped to the **entire bucket**, with the current version action transitioning objects to Glacier Instant Retrieval.

![Lifecycle Rules List](./screenshots/lifecycle-rules-list.png)

---

### 8. Lifecycle Rule — Full Transition Timeline
Detailed view of the `LawFirmArchive` rule showing all configured transitions:
- **Day 0**: Objects uploaded
- **Day 90**: Move to Glacier Instant Retrieval
- **Day 365**: Move to Glacier Flexible Retrieval (formerly Glacier)
- **Day 2555** (~7 years): Objects expire (permanent deletion)

![Lifecycle Rule Detail](./screenshots/lifecycle-rule-detail.png)

---

## 🚀 Getting Started

### Prerequisites
- AWS CLI v2 configured with appropriate IAM credentials
- Python 3.8+ with `boto3` installed
- Access to the firm's AWS account (`lagos-lawfirm` org)

### Clone & Setup
```bash
git clone https://github.com/your-org/lagos-lawfirm-s3-dms.git
cd lagos-lawfirm-s3-dms
pip install -r requirements.txt
```

### Deploy Bucket & Policies
```bash
# Create the S3 bucket
aws s3api create-bucket \
  --bucket lagos-lawfirm-docs-simeonjnr \
  --region af-south-1 \
  --create-bucket-configuration LocationConstraint=af-south-1

# Apply bucket policy
aws s3api put-bucket-policy \
  --bucket lagos-lawfirm-docs-simeonjnr \
  --policy file://policies/bucket-policy.json

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket lagos-lawfirm-docs-simeonjnr \
  --versioning-configuration Status=Enabled

# Apply lifecycle rules
aws s3api put-bucket-lifecycle-configuration \
  --bucket lagos-lawfirm-docs-simeonjnr \
  --lifecycle-configuration file://policies/lifecycle.json
```

---

## 📁 Repository Structure

```
lagos-lawfirm-s3-dms/
├── README.md
├── policies/
│   ├── bucket-policy.json
│   ├── lifecycle.json
│   └── iam-roles/
│       ├── admin-role.json
│       ├── paralegal-role.json
│       └── readonly-role.json
├── scripts/
│   ├── generate_presigned_url.py
│   ├── upload_document.py
│   └── audit_access_logs.py
└── screenshots/
    ├── s3-bucket-overview.png
    ├── versioning-encryption.png
    ├── lifecycle-rules.png
    ├── iam-roles.png
    ├── bucket-policy.png
    ├── cloudtrail-logs.png
    └── presigned-url-test.png
```

*Built for secure legal document management | AWS S3 · KMS · IAM · CloudTrail*
