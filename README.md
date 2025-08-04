# AWS Security Services Integration with SIEM

This document provides a step-by-step guide to integrate various **AWS Security Services**—**GuardDuty**, **CloudTrail**, **WAF**, and **VPC Flow Logs**—with a **SIEM** system. Both **pull (S3-based)** and **push (EventBridge/Lambda)** approaches are included.

---

## 🧹 Integration Overview

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

* ✅ **Your Role**: Create and configure multi-region trail
* Actions:

  * Create a trail for all regions
  * Enable management events, data events (optional), and insights (optional)
  * Destination: S3 bucket (e.g., `my-org-logs/cloudtrail/`)
* **IAM Role Required**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["cloudtrail:CreateTrail", "cloudtrail:StartLogging", "s3:PutObject"],
      "Resource": "*"
    }
  ]
}
```

### GuardDuty

* ✅ **Your Role**: Enable and configure export
* Actions:

  * Enable in all regions
  * Use "Export Findings" to send to CloudWatch or EventBridge
* **IAM Role Required**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["guardduty:EnableOrganizationAdminAccount", "events:PutRule", "events:PutTargets"],
      "Resource": "*"
    }
  ]
}
```

### AWS WAF

* ✅ **Your Role**: Set up logging through Firehose
* Actions:

  * Enable logging via WAF console
  * Destination: Kinesis Data Firehose → S3 (or EventBridge)
* **IAM Role Required**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["waf:PutLoggingConfiguration", "firehose:PutRecord", "firehose:PutRecordBatch"],
      "Resource": "*"
    }
  ]
}
```

### VPC Flow Logs

* ✅ **Your Role**: Create flow logs and set destination
* Actions:

  * Create flow logs per VPC or ENI
  * Destination: CloudWatch Logs or S3 bucket (e.g., `my-org-logs/vpc-flow-logs/`)
* **IAM Role Required**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ec2:CreateFlowLogs", "logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"],
      "Resource": "*"
    }
  ]
}
```

---

## 🔹 Step 2: Centralize Logs in S3 (Pull Model)

| Service       | S3 Prefix Path           |
| ------------- | ------------------------ |
| CloudTrail    | `cloudtrail/`            |
| GuardDuty     | `guardduty/`             |
| WAF           | `waf-logs/` via Firehose |
| VPC Flow Logs | `vpc-flow-logs/`         |

**✅ Your Role: Create S3 bucket and IAM policy for SIEM access**

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

✅ **Your Role**: Create rules and targets

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

* **WAF Logs**: Use Firehose S3 delivery + Event Notification or CloudWatch filter
* **VPC Flow Logs**: CloudWatch Logs subscription filter

### b. Lambda Function Example

✅ **Your Role**: Create Lambda and configure EventBridge trigger

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

✅ **Your Role**: Attach appropriate IAM policies

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "events:PutEvents"
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


## 🔐 Best Practices

* Enable logging across all regions and accounts
* Centralize logs using AWS Organizations
* Encrypt logs with SSE-KMS
* Use least privilege IAM roles
* Apply lifecycle policies to archive/delete old logs
* Monitor Lambda/Firehose errors via CloudWatch

---

## 🔎 Additional Note on GuardDuty Findings

If you have **300+ GuardDuty findings**, you can:

* Export all findings using **Amazon EventBridge → Lambda → SIEM** for real-time detection
* Or schedule periodic export to S3 and ingest using **SIEM pull model**
* Consider setting up **severity-based alert rules** in your SIEM

---

If you'd like help exporting this as a `.md` file, generating a diagram, or writing the Lambda code, let me know!
