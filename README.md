# AWS Security Services Integration with SIEM

This document provides a step-by-step guide to integrate various **AWS Security Services**—**GuardDuty**, **CloudTrail**, **WAF**, and **VPC Flow Logs**—with a **SIEM** system. Both **pull (S3-based)** and **push (EventBridge/Lambda)** approaches are included.

---

## 🧩 Integration Overview

| Component        | Purpose                                   |
| ---------------- | ----------------------------------------- |
| AWS CloudTrail   | Logs all API calls in your account        |
| Amazon GuardDuty | Detects threats using threat intelligence |
| AWS WAF          | Logs web access control events            |
| VPC Flow Logs    | Captures network traffic metadata         |
| Amazon S3        | Centralized log storage                   |
| EventBridge      | Detects events in real-time               |
| AWS Lambda       | Forwards logs to SIEM                     |
| SIEM             | Receives, parses, and analyzes logs       |

---

## 🔹 Step 1: Enable Logging for AWS Services

### CloudTrail

* Create a trail for all regions
* Enable management events, data events (optional), and insights (optional)
* Destination: S3 bucket (e.g., `my-org-logs/cloudtrail/`)

### GuardDuty

* Enable in all regions
* Use "Export Findings" to send to CloudWatch or EventBridge

### AWS WAF

* Enable logging via WAF console
* Destination: Kinesis Data Firehose → S3 (or EventBridge)

### VPC Flow Logs

* Create flow logs per VPC or ENI
* Destination: CloudWatch Logs or S3 bucket (e.g., `my-org-logs/vpc-flow-logs/`)

---

## 🔹 Step 2: Centralize Logs in S3 (Pull Model)

| Service       | S3 Prefix Path           |
| ------------- | ------------------------ |
| CloudTrail    | `cloudtrail/`            |
| GuardDuty     | `guardduty/`             |
| WAF           | `waf-logs/` via Firehose |
| VPC Flow Logs | `vpc-flow-logs/`         |

**IAM Policy for SIEM:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::my-org-logs",
        "arn:aws:s3:::my-org-logs/*"
      ]
    }
  ]
}
```

**SIEMs Supported**

* Splunk (Add-ons)
* QRadar (S3/CloudWatch)
* ELK (Filebeat/Logstash)
* Azure Sentinel (Logic Apps, Lambda bridge)

---

## 🔹 Step 3: Push Mode with EventBridge + Lambda

### a. EventBridge Rules

Create one rule per service if needed:

* **GuardDuty Findings**

```json
{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Finding"]
}
```

* **CloudTrail Events**

```json
{
  "source": ["aws.cloudtrail"]
}
```

* **WAF Logs via Firehose to S3 → Event Notification**
* **VPC Flow Logs via CloudWatch Logs Subscriptions**

### b. Lambda Function Example

```python
import json
import requests

SIEM_ENDPOINT = "https://siem.example.com/ingest"

def lambda_handler(event, context):
    print(json.dumps(event))
    response = requests.post(SIEM_ENDPOINT, json=event)
    return {"status": response.status_code}
```

### c. IAM Role for Lambda

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 🔹 Step 4: Alerting & Monitoring

| Use Case                  | Detection Source        |
| ------------------------- | ----------------------- |
| API Misuse                | CloudTrail              |
| Port Scanning             | GuardDuty/VPC Flow Logs |
| Malicious IP Access       | WAF or GuardDuty        |
| Excessive 4xx/5xx Traffic | WAF                     |

---

## ✅ Architecture Diagram

```plaintext
AWS Services → S3 (Centralized Logs) ──────► SIEM (Pull)
                 │
                 └─► EventBridge → Lambda → SIEM (Push)
```

---

## 🔐 Best Practices

* Enable logging across all regions and accounts
* Centralize logs using AWS Organizations
* Encrypt logs with SSE-KMS
* Use least privilege IAM roles
* Apply lifecycle policies to archive/delete old logs
* Monitor Lambda/Firehose errors via CloudWatch
