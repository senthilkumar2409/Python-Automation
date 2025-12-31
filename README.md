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
- [Test Events](#test-events)
- [Output Example](#output-example)
- [Deployment](#deployment)
- [Monitoring](#monitoring)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)
- [License](#license)

## 🎯 Overview

This Lambda function provides **automated encryption compliance auditing** for AWS resources. It scans all EBS volumes and S3 buckets in your AWS account, identifies unencrypted resources, and generates comprehensive compliance reports with actionable recommendations.

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

```
┌──────────────────┐
│   EventBridge    │
│  (Cron Schedule) │
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────────┐
│    AWS Lambda Function            │
│ encryption-compliance-checker     │
├──────────────────────────────────┤
│  -  check_ebs_encryption()         │
│  -  check_s3_encryption()          │
│  -  generate_report()              │
│  -  send_notification() (optional) │
└────────┬─────────────┬────────────┘
         │             │
         ▼             ▼
    ┌────────┐    ┌──────────┐
    │   EC2  │    │    S3    │
    │ Service│    │ Service  │
    └────────┘    └──────────┘
         │             │
         └─────┬───────┘
               ▼
        ┌────────────────┐
        │  CloudWatch    │
        │     Logs       │
        └────────────────┘
```

## 📋 Prerequisites

### AWS Services Required
- **AWS Lambda** - Serverless compute
- **AWS EC2** - For EBS volume scanning
- **AWS S3** - For bucket scanning
- **AWS CloudWatch** - For logging and monitoring
- **AWS EventBridge** (optional) - For scheduled triggers
- **AWS SNS** (optional) - For notifications

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
Environment Variables: (optional)
  - SEND_NOTIFICATIONS: false
  - SNS_TOPIC_ARN: arn:aws:sns:region:account:topic-name
```

### EventBridge Schedule (Optional)

```bash
# Create scheduled rule
Rule Name: encryption-compliance-daily
Schedule: cron(0 9 * * ? *)  # Daily at 9 AM UTC
Target: Lambda function (encryption-compliance-checker)
```

## 📖 Usage

### Invoke via EventBridge (Automated)

Function is automatically invoked on the configured schedule. No manual action required.

### Invoke via AWS Console (Manual)

1. Open Lambda function
2. Click **Test** tab
3. Select or create test event
4. Click **Test**
5. View results in execution result

### Invoke via AWS CLI

```bash
# Invoke function
aws lambda invoke \
  --function-name encryption-compliance-checker \
  --region us-east-1 \
  response.json

# View response
cat response.json
```

### Invoke via Python (Local Testing)

```python
import boto3
import json

lambda_client = boto3.client('lambda')

response = lambda_client.invoke(
    FunctionName='encryption-compliance-checker',
    InvocationType='RequestResponse',
    Payload=json.dumps({
        'source': 'manual-test',
        'action': 'check-encryption'
    })
)

result = json.loads(response['Payload'].read())
print(json.dumps(result, indent=2))
```

## 🧪 Test Events

### Test Event 1: EventBridge Scheduled Event

```json
{
  "version": "0",
  "id": "6a7e8feb-b491-4cf7-a9f1-bf3703467718",
  "detail-type": "Scheduled Event",
  "source": "aws.events",
  "account": "123456789012",
  "time": "2025-12-31T20:30:00Z",
  "region": "us-east-1",
  "resources": [
    "arn:aws:events:us-east-1:123456789012:rule/encryption-compliance-check"
  ],
  "detail": {}
}
```

### Test Event 2: Custom Manual Trigger

```json
{
  "source": "manual-test",
  "action": "check-encryption",
  "timestamp": "2025-12-31T20:30:00Z",
  "region": "us-east-1"
}
```

### Test Event 3: Minimal Test

```json
{
  "test": true
}
```

## 📊 Output Example

### Console Output

```
Lambda invoked at 2025-12-31T20:30:00.123456
Event: {...}

============================================================
CHECKING EBS VOLUMES ENCRYPTION
============================================================
Total EBS volumes found: 3

⚠️  UNENCRYPTED EBS VOLUME DETECTED
  Volume ID: vol-0123456789abcdef0
  Name: Production-DB
  Size: 100 GB
  Type: gp3
  State: in-use
  AZ: us-east-1a
  Encrypted: False

✅ Volume vol-abcdefghij123456 (Backup) is encrypted

============================================================
CHECKING S3 BUCKETS ENCRYPTION
============================================================
Total S3 buckets found: 5

⚠️  UNENCRYPTED S3 BUCKET DETECTED
  Bucket Name: legacy-data-bucket
  Region: us-east-1
  Created: 2023-05-10T08:15:00
  Encrypted: False

✅ Bucket my-encrypted-bucket has encryption enabled

============================================================
ENCRYPTION COMPLIANCE SUMMARY
============================================================
Timestamp: 2025-12-31T20:30:00.123456
Total Unencrypted Resources: 2
  - Unencrypted EBS Volumes: 1
  - Unencrypted S3 Buckets: 1
Compliance Status: NON-COMPLIANT
============================================================
```

### JSON Response

```json
{
  "statusCode": 200,
  "body": {
    "timestamp": "2025-12-31T20:30:00.123456",
    "summary": {
      "totalUnencryptedResources": 2,
      "unencryptedEBS": 1,
      "unencryptedS3": 1,
      "complianceStatus": "NON-COMPLIANT"
    },
    "unencryptedEBSVolumes": [
      {
        "resourceId": "vol-0123456789abcdef0",
        "resourceType": "EBS Volume",
        "name": "Production-DB",
        "size": "100 GB",
        "volumeType": "gp3",
        "state": "in-use",
        "availabilityZone": "us-east-1a",
        "encrypted": false,
        "message": "⚠️  UNENCRYPTED EBS VOLUME DETECTED",
        "severity": "HIGH",
        "recommendation": "Create encrypted snapshot and restore to encrypted volume",
        "timestamp": "2025-12-31T20:30:00.123456"
      }
    ],
    "unencryptedS3Buckets": [
      {
        "resourceId": "legacy-data-bucket",
        "resourceType": "S3 Bucket",
        "name": "legacy-data-bucket",
        "region": "us-east-1",
        "creationDate": "2023-05-10T08:15:00",
        "encrypted": false,
        "message": "⚠️  UNENCRYPTED S3 BUCKET DETECTED",
        "severity": "HIGH",
        "recommendation": "Enable bucket encryption (SSE-S3 or SSE-KMS)",
        "timestamp": "2025-12-31T20:30:00.123456"
      }
    ],
    "nextSteps": [
      "Review all unencrypted resources",
      "Enable encryption for all resources",
      "Set encryption policies to enforce encryption by default",
      "Schedule regular compliance audits"
    ]
  }
}
```

## 🔧 Deployment

### Deploy via AWS Console

1. Open Lambda function
2. Copy Python code into editor
3. Click **Deploy**
4. Function is live and ready to use

### Deploy via AWS CLI

```bash
# Create deployment package
zip lambda.zip index.py

# Deploy
aws lambda update-function-code \
  --function-name encryption-compliance-checker \
  --zip-file fileb://lambda.zip \
  --region us-east-1
```

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

## 📡 Monitoring

### CloudWatch Logs

```bash
# View logs
aws logs tail /aws/lambda/encryption-compliance-checker --follow

# Search for errors
aws logs filter-log-events \
  --log-group-name /aws/lambda/encryption-compliance-checker \
  --filter-pattern "ERROR"
```

### CloudWatch Metrics

Monitor:
- **Duration** - Function execution time
- **Errors** - Failed invocations
- **Throttles** - Rate limiting
- **ConcurrentExecutions** - Concurrent runs

### CloudWatch Dashboard

Create custom dashboard to track:
- Total unencrypted resources
- Compliance status trends
- Scan execution frequency
- Error rates

## 🔍 Troubleshooting

### Issue: Permission Denied

**Error:** `User: arn:aws:iam::... is not authorized to perform: ec2:DescribeVolumes`

**Solution:**
```bash
# Ensure Lambda execution role has correct policy
# Attach AmazonEC2ReadOnlyAccess and S3ReadOnlyAccess
```

### Issue: Lambda Timeout

**Error:** `Task timed out after 60.00 seconds`

**Solution:**
```bash
# Increase timeout in Lambda configuration
# Configuration → General Configuration → Timeout → Set to 120 seconds
```

### Issue: No Resources Found

**Error:** `Total EBS volumes found: 0`

**Solution:**
- Verify resources exist in the Lambda region
- Check IAM permissions for resource access
- Verify AWS credentials are correct

### Issue: S3 Encryption Check Fails

**Error:** `Error checking bucket: NoSuchBucket`

**Solution:**
- Verify S3 bucket names are correct
- Check Lambda IAM policy includes S3 permissions
- Ensure bucket region is accessible

## ✅ Best Practices

1. **Regular Audits**
   - Schedule compliance checks daily or weekly
   - Monitor trends over time

2. **Encryption Standards**
   - Use AWS KMS for sensitive data (SSE-KMS)
   - Use SSE-S3 for general compliance

3. **Automated Response**
   - Enable SNS notifications for alerts
   - Create automated remediation workflows

4. **Documentation**
   - Document exceptions and approval chains
   - Maintain audit logs for compliance

5. **Testing**
   - Test function with sample data before production
   - Verify IAM permissions in dev environment first

6. **Cost Optimization**
   - Schedule compliance checks during off-peak hours
   - Use Lambda reserved concurrency if needed

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

## 📝 License

MIT License - Free to use and modify

## 🤝 Contributing

Feel free to fork and submit pull requests for improvements.

## 📞 Support

For issues or questions:
1. Check CloudWatch Logs for detailed error messages
2. Verify IAM permissions are correct
3. Test with sample events in Lambda console
4. Review AWS documentation for service limits

---

**Last Updated:** December 31, 2025  
**Author:** DevOps Team  
**Status:** Production Ready ✅
```

This README provides comprehensive documentation covering all aspects of the encryption compliance checker function. You can customize it further based on your specific organizational needs.
