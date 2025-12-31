# Python Scripting (Automation) - Compliance check

# AWS Encryption Compliance Checker

A serverless Lambda function that automatically scans your AWS infrastructure to identify **unencrypted EBS volumes** and **S3 buckets**, providing detailed compliance reports and recommendations.

## 📋 Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [Output Example](#output-example)
- [Deployment](#deployment)

## 🎯 Overview

This Lambda function provides **automated encryption compliance auditing** for AWS resources. It scans all EBS volumes and S3 buckets in your AWS account, identifies unencrypted resources, and generates comprehensive compliance reports with actionable recommendations.

**For example:** If we found the EBS volumes or S3 Buckets are Not encrypted, Lambda function will triggers the SNS notification(outlook) to notify the team.

**Use Case:** Ensure all critical infrastructure meets encryption security standards and compliance requirements (SOC 2, HIPAA, PCI-DSS, etc.).

## ✨ Features

- ✅ **EBS Volume Scanning** - Detects unencrypted EBS volumes across all regions
- ✅ **S3 Bucket Scanning** - Identifies S3 buckets without server-side encryption
- ✅ **Detailed Reporting** - Provides comprehensive resource information and recommendations
- ✅ **CloudWatch Logging** - Logs all findings with visual indicators
- ✅ **Automated Scheduling** - Integrates with EventBridge for scheduled compliance checks
- ✅ **Error Handling** - Robust exception handling with meaningful error messages
- ✅ **Type Hints** - Python 3.13+ compatible with full type annotations
- ✅ **Modular Design** - Reusable functions for testing and extension

## 🏗️ Architecture



## 📚 Tech Stack

| Technology | Version | Purpose |
|-----------|---------|---------|
| Python    | 3.13+   | Lambda runtime language |
| Boto3     | Latest  | AWS SDK for Python |
| AWS Lambda| -       | Serverless compute |
| AWS EC2   | -       | EBS volume service |
| AWS S3    | -       | Object storage service |
| CloudWatch| -       | Logging & monitoring |
| EventBridge | -     | Scheduled triggers |


## 📋 Prerequisites

### AWS Services Required
- **AWS Lambda** - Serverless compute
- **AWS EC2** - For EBS volume scanning
- **AWS S3** - For bucket scanning
- **AWS CloudWatch** - For logging and monitoring
- **AWS EventBridge**  - For scheduled triggers
- **AWS SNS**  - For notifications

### Local Development Tools
- Python 3.13+
- AWS CLI configured with appropriate credentials
- Boto3 library
- IAM user/role with necessary permissions

### IAM Permissions Required

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeVolumes",
        "s3:ListAllMyBuckets",
        "s3:GetBucketEncryption",
        "s3:GetBucketLocation"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
```

## 🚀 Installation

### Step 1: Create Lambda Function

```bash
# Via AWS Console
1. Go to AWS Lambda → Create Function
2. Function name: encryption-compliance-checker
3. Runtime: Python 3.13
4. Create function
```

### Step 2: Configure IAM Role

```bash
# Attach policy to Lambda execution role
1. Go to IAM → Roles
2. Select Lambda execution role
3. Attach the IAM policy from Prerequisites section
```

### Step 3: Upload Function Code

```bash
# Copy the Python code into Lambda editor
# Or upload as ZIP file with:
# - lambda_function.py (main code)
# - requirements.txt (if using external libraries)
```

### Step 4: Set Runtime Configuration

- **Handler:** `index.lambda_handler`
- **Timeout:** 60 seconds (recommended)
- **Memory:** 512 MB (recommended)

## ⚙️ Configuration

### Lambda Function Configuration

```yaml
Function Name: encryption-compliance-checker
Runtime: Python 3.13
Handler: index.lambda_handler
Timeout: 60 seconds
Memory: 512 MB
Ephemeral Storage: 512 MB
Environment Variables: 
  - SEND_NOTIFICATIONS: true
  - SNS_TOPIC_ARN: arn:aws:sns:us-east-1:490004609243:SNS
```

### EventBridge Schedule 

```bash
# Create scheduled rule
Rule Name: encryption-compliance-daily
Schedule: cron(0 9 * * ? *)  # Daily at 1 AM UTC
Target: Lambda function (encryption-compliance-checker)
```

## 📖 Usage

### Invoke via EventBridge (Automated)

Function is automatically invoked on the configured schedule. No manual action required.


### Lambda function (encryption-compliance-checker)
```python
import os
import boto3
import json
import urllib.request
from datetime import datetime
from typing import Dict, Any, List
from botocore.exceptions import ClientError

# Initialize AWS clients (use Lambda region by default)
ec2_client = boto3.client("ec2")
s3_client = boto3.client("s3")
sns_client = boto3.client("sns")

SNS_TOPIC_ARN = os.getenv("SNS_TOPIC_ARN", "")   # required for SNS notify
SLACK_WEBHOOK_URL = os.getenv("SLACK_WEBHOOK_URL", "")  # required for Slack notify


def send_slack_notification(message: str) -> None:
    """Send Slack message via Incoming Webhook."""
    if not SLACK_WEBHOOK_URL:
        print("⚠️ SLACK_WEBHOOK_URL not set; skipping Slack notification")
        return

    payload = {"text": message}
    data = json.dumps(payload).encode("utf-8")

    req = urllib.request.Request(
        SLACK_WEBHOOK_URL,
        data=data,
        headers={"Content-Type": "application/json"},
        method="POST",
    )

    try:
        with urllib.request.urlopen(req, timeout=10) as resp:
            print(f"✅ Slack notified. HTTP status={resp.status}")
    except Exception as e:
        print(f"⚠️ Failed Slack notification: {str(e)}")


def lambda_handler(event: Dict[str, Any], context: Any) -> Dict[str, Any]:
    try:
        print(f"Lambda invoked at {datetime.now().isoformat()}")
        print(f"Event: {json.dumps(event)}")

        unencrypted_ebs = check_ebs_encryption()
        unencrypted_s3 = check_s3_encryption()

        report = generate_report(unencrypted_ebs, unencrypted_s3)

        # ✅ Send Slack (always: success or non-compliant)
        send_slack_for_report(report)

        # ✅ Send SNS (only when non-compliant)
        send_notification(report)

        return {"statusCode": 200, "body": json.dumps(report, indent=2)}

    except Exception as e:
        error_message = f"Error checking encryption status: {str(e)}"
        print(f"ERROR: {error_message}")

        # Optional: Slack on failure
        send_slack_notification(f"⚠️ Encryption audit Lambda failed: {error_message}")

        return {
            "statusCode": 500,
            "body": json.dumps({"error": error_message, "timestamp": datetime.now().isoformat()}),
        }


def check_ebs_encryption() -> List[Dict[str, Any]]:
    """Check all EBS volumes for encryption status (paginates)."""
    unencrypted_volumes: List[Dict[str, Any]] = []

    print("\n" + "=" * 60)
    print("CHECKING EBS VOLUMES ENCRYPTION")
    print("=" * 60)

    # Use paginator so you don't miss volumes on large accounts. [web:144][web:302]
    paginator = ec2_client.get_paginator("describe_volumes")
    total_seen = 0

    for page in paginator.paginate():
        for volume in page.get("Volumes", []):
            total_seen += 1

            volume_id = volume["VolumeId"]
            is_encrypted = volume.get("Encrypted", False)  # EBS encryption flag [web:144]
            volume_size = volume.get("Size")
            volume_type = volume.get("VolumeType")
            state = volume.get("State")
            availability_zone = volume.get("AvailabilityZone")

            tags = {tag["Key"]: tag["Value"] for tag in volume.get("Tags", [])}
            volume_name = tags.get("Name", "No Name")

            if not is_encrypted:
                volume_details = {
                    "resourceId": volume_id,
                    "resourceType": "EBS Volume",
                    "name": volume_name,
                    "size": f"{volume_size} GB",
                    "volumeType": volume_type,
                    "state": state,
                    "availabilityZone": availability_zone,
                    "encrypted": is_encrypted,
                    "message": "⚠️ UNENCRYPTED EBS VOLUME DETECTED",
                    "severity": "HIGH",
                    "recommendation": "Create encrypted snapshot and restore to encrypted volume",
                    "timestamp": datetime.now().isoformat(),
                }
                unencrypted_volumes.append(volume_details)
                print(f"⚠️ Unencrypted EBS: {volume_id} ({volume_name})")
            else:
                print(f"✅ EBS encrypted: {volume_id} ({volume_name})")

    print(f"\nTotal EBS volumes scanned: {total_seen}")
    print(f"Total unencrypted EBS volumes: {len(unencrypted_volumes)}")
    return unencrypted_volumes


def check_s3_encryption() -> List[Dict[str, Any]]:
    """Check all S3 buckets for default encryption config."""
    unencrypted_buckets: List[Dict[str, Any]] = []

    print("\n" + "=" * 60)
    print("CHECKING S3 BUCKETS ENCRYPTION (DEFAULT ENCRYPTION)")
    print("=" * 60)

    buckets = s3_client.list_buckets().get("Buckets", [])
    print(f"Total S3 buckets found: {len(buckets)}")

    for bucket in buckets:
        bucket_name = bucket["Name"]
        creation_date = bucket.get("CreationDate").isoformat() if bucket.get("CreationDate") else "Unknown"

        try:
            # If bucket has default encryption, this returns config; otherwise it errors. [web:136][web:143]
            enc = s3_client.get_bucket_encryption(Bucket=bucket_name)
            rules = enc.get("ServerSideEncryptionConfiguration", {}).get("Rules", [])
            if not rules:
                raise ClientError(
                    {"Error": {"Code": "ServerSideEncryptionConfigurationNotFoundError", "Message": "No Rules"}},
                    "GetBucketEncryption",
                )

            print(f"✅ S3 encrypted by default: {bucket_name}")

        except ClientError as e:
            # Treat missing encryption config as non-compliant. [web:143]
            try:
                loc = s3_client.get_bucket_location(Bucket=bucket_name)
                region = loc.get("LocationConstraint") or "us-east-1"
            except Exception:
                region = "Unknown"

            bucket_details = {
                "resourceId": bucket_name,
                "resourceType": "S3 Bucket",
                "name": bucket_name,
                "region": region,
                "creationDate": creation_date,
                "encrypted": False,
                "message": "⚠️ UNENCRYPTED S3 BUCKET DETECTED (no default encryption)",
                "severity": "HIGH",
                "recommendation": "Enable bucket default encryption (SSE-S3 or SSE-KMS)",
                "timestamp": datetime.now().isoformat(),
                "errorCode": e.response.get("Error", {}).get("Code"),
            }
            unencrypted_buckets.append(bucket_details)
            print(f"⚠️ S3 bucket without default encryption: {bucket_name} (region={region})")

    print(f"\nTotal unencrypted S3 buckets: {len(unencrypted_buckets)}")
    return unencrypted_buckets


def generate_report(unencrypted_ebs: List[Dict[str, Any]], unencrypted_s3: List[Dict[str, Any]]) -> Dict[str, Any]:
    total_unencrypted = len(unencrypted_ebs) + len(unencrypted_s3)

    report = {
        "timestamp": datetime.now().isoformat(),
        "summary": {
            "totalUnencryptedResources": total_unencrypted,
            "unencryptedEBS": len(unencrypted_ebs),
            "unencryptedS3": len(unencrypted_s3),
            "complianceStatus": "COMPLIANT" if total_unencrypted == 0 else "NON-COMPLIANT",
        },
        "unencryptedEBSVolumes": unencrypted_ebs,
        "unencryptedS3Buckets": unencrypted_s3,
        "nextSteps": [
            "Review all unencrypted resources",
            "Enable encryption for all resources",
            "Set encryption policies to enforce encryption by default",
            "Schedule regular compliance audits",
        ],
    }

    print("\n" + "=" * 60)
    print("ENCRYPTION COMPLIANCE SUMMARY")
    print("=" * 60)
    print(f"Timestamp: {report['timestamp']}")
    print(f"Total Unencrypted Resources: {total_unencrypted}")
    print(f"  - Unencrypted EBS Volumes: {len(unencrypted_ebs)}")
    print(f"  - Unencrypted S3 Buckets: {len(unencrypted_s3)}")
    print(f"Compliance Status: {report['summary']['complianceStatus']}")
    print("=" * 60 + "\n")

    return report


def send_slack_for_report(report: Dict[str, Any]) -> None:
    """Slack: success message if compliant, else list resource IDs."""
    total_unencrypted = report["summary"]["totalUnencryptedResources"]

    if total_unencrypted == 0:
        send_slack_notification("EBS and S3 resources encrypted ✅")
        return

    lines = []
    lines.append("🚨 *AWS Encryption NON-COMPLIANT*")
    lines.append(f"*Total unencrypted:* {total_unencrypted}")
    lines.append(f"*EBS unencrypted:* {report['summary']['unencryptedEBS']}")
    lines.append(f"*S3 unencrypted:* {report['summary']['unencryptedS3']}")

    if report["unencryptedEBSVolumes"]:
        lines.append("\n*Unencrypted EBS Volume IDs:*")
        for v in report["unencryptedEBSVolumes"][:25]:
            lines.append(f"- {v['resourceId']} ({v.get('name', 'No Name')})")

    if report["unencryptedS3Buckets"]:
        lines.append("\n*S3 Buckets without default encryption:*")
        for b in report["unencryptedS3Buckets"][:25]:
            lines.append(f"- {b['resourceId']} (region={b.get('region')})")

    if report["summary"]["unencryptedEBS"] > 25 or report["summary"]["unencryptedS3"] > 25:
        lines.append("\n(Showing first 25 of each. Check CloudWatch logs for full list.)")

    send_slack_notification("\n".join(lines))


def send_notification(report: Dict[str, Any]) -> None:
    """SNS: send only if unencrypted resources found."""
    total_unencrypted = report["summary"]["totalUnencryptedResources"]
    if total_unencrypted <= 0:
        print("✅ No unencrypted resources; SNS not sent")
        return

    if not SNS_TOPIC_ARN:
        print("⚠️ SNS_TOPIC_ARN not set; skipping SNS notification")
        return

    message = (
        "AWS ENCRYPTION COMPLIANCE ALERT\n\n"
        f"Total Unencrypted Resources: {total_unencrypted}\n"
        f"- Unencrypted EBS Volumes: {report['summary']['unencryptedEBS']}\n"
        f"- Unencrypted S3 Buckets: {report['summary']['unencryptedS3']}\n\n"
        f"Status: {report['summary']['complianceStatus']}\n\n"
        "Please review and enable encryption for all resources.\n"
    )

    # SNS publish signature: TopicArn + Message (+ Subject optional). [web:291]
    resp = sns_client.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject="AWS Encryption Compliance Alert",
        Message=message,
    )
    print(f"✅ SNS notification sent. MessageId={resp.get('MessageId')}")
```


### Function Logs:

```
Lambda invoked at 2025-12-31T20:30:00.123456
Event: {...}

Function Logs:
START RequestId: 8a67c8e2-f956-4186-a8b1-b8a1757c52e2 Version: $LATEST
Lambda invoked at 2025-12-31T16:49:19.547292
Event: {"test": true, "environment": "development"}
============================================================
CHECKING EBS VOLUMES ENCRYPTION
============================================================
⚠️ Unencrypted EBS: vol-0b363ae31d34c48ef (No Name)
⚠️ Unencrypted EBS: vol-0d06520196f3e9768 (No Name)
Total EBS volumes scanned: 2
Total unencrypted EBS volumes: 2
============================================================
CHECKING S3 BUCKETS ENCRYPTION (DEFAULT ENCRYPTION)
============================================================
Total S3 buckets found: 1
✅ S3 encrypted by default: uno-velero
Total unencrypted S3 buckets: 0
============================================================
ENCRYPTION COMPLIANCE SUMMARY
============================================================
Timestamp: 2025-12-31T16:49:20.052263
Total Unencrypted Resources: 2
- Unencrypted EBS Volumes: 2
- Unencrypted S3 Buckets: 0
Compliance Status: NON-COMPLIANT
============================================================
⚠️ SLACK_WEBHOOK_URL not set; skipping Slack notification
✅ SNS notification sent. MessageId=0acbc41c-e0ad-5876-bc1b-a5d3b6fdda76
```


## 🔧 Deployment

### Deploy via AWS Console

1. Open Lambda function
2. Copy Python code into editor
3. Click **Deploy**
4. Function is live and ready to use


### Deploy via Terraform

```hcl
resource "aws_lambda_function" "encryption_checker" {
  filename      = "lambda.zip"
  function_name = "encryption-compliance-checker"
  role          = aws_iam_role.lambda_role.arn
  handler       = "index.lambda_handler"
  runtime       = "python3.13"
  timeout       = 60
  memory_size   = 512
}
```

