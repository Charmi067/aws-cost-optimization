# 💰 AWS Cost Optimization — Automated EBS Snapshot Cleanup

> A serverless automation project that identifies and deletes orphaned EBS snapshots using **AWS Lambda + EventBridge Scheduler**, reducing unnecessary AWS storage costs with zero manual intervention.

---

## 📸 Screenshots

### Lambda Function — Live Execution
![Lambda Function Execution](./screenshots/lambda_execution.jpeg)
> *Real execution log showing snapshot `snap-0a1920f1e46646372` deleted as its associated volume was not attached to any running instance. Duration: 821ms, Memory: 101MB.*

### EventBridge Scheduler — Enabled & Running
![EventBridge Scheduler](./screenshots/eventbridge_scheduler.jpeg)
> *EventBridge Scheduler `ebs-snapshots-lambda` configured and enabled, targeting the Lambda function on a recurring schedule. Timezone: Asia/Calcutta.*

---

## 🏗️ Architecture

```
EventBridge Scheduler (Cron)
         │
         │  Triggers on schedule
         ▼
AWS Lambda Function
(cost-optimization-EBS-snapshots)
         │
         ├── List all EBS Snapshots (boto3 / EC2 API)
         │
         ├── For each snapshot:
         │     ├── Check if associated volume exists
         │     ├── Check if volume is attached to a running EC2 instance
         │     └── If orphaned → DELETE snapshot
         │
         └── Log results to CloudWatch
```

---

## ✨ What It Does

- 🔍 **Scans** all EBS snapshots in the AWS account
- 🧠 **Identifies orphaned snapshots** — volumes that no longer exist or are not attached to any running EC2 instance
- 🗑️ **Auto-deletes** unnecessary snapshots to eliminate storage costs
- ⏰ **Runs on a schedule** via EventBridge Scheduler — fully automated, zero manual steps
- 📋 **Logs every action** to CloudWatch for audit trail

---

## 🛠️ Tech Stack

| Component | Service |
|---|---|
| Automation Logic | AWS Lambda (Python) |
| Scheduling | Amazon EventBridge Scheduler |
| AWS SDK | boto3 (EC2 API) |
| Logging | AWS CloudWatch Logs |
| IAM | Least-privilege execution role |

---

## ⚙️ Setup & Deployment

### Prerequisites
- AWS Account with appropriate IAM permissions
- Python 3.x (for Lambda runtime)

### IAM Permissions Required

The Lambda execution role needs:
```json
{
  "Effect": "Allow",
  "Action": [
    "ec2:DescribeSnapshots",
    "ec2:DescribeVolumes",
    "ec2:DescribeInstances",
    "ec2:DeleteSnapshot"
  ],
  "Resource": "*"
}
```

### Lambda Function Core Logic

```python
import boto3

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')
    
    # Get all snapshots owned by this account
    response = ec2.describe_snapshots(OwnerIds=['self'])
    
    for snapshot in response['Snapshots']:
        snapshot_id = snapshot['SnapshotId']
        volume_id = snapshot.get('VolumeId')
        
        try:
            # Check if the volume still exists
            vol = ec2.describe_volumes(VolumeIds=[volume_id])
            
            # Check if volume is attached to a running instance
            attachments = vol['Volumes'][0].get('Attachments', [])
            if not attachments:
                ec2.delete_snapshot(SnapshotId=snapshot_id)
                print(f"Deleted EBS snapshot {snapshot_id} as its volume was not attached to any running instance.")
                
        except ec2.exceptions.ClientError as e:
            if e.response['Error']['Code'] == 'InvalidVolume.NotFound':
                # Volume no longer exists — delete the orphaned snapshot
                ec2.delete_snapshot(SnapshotId=snapshot_id)
                print(f"Deleted EBS snapshot {snapshot_id} as its associated volume was not found.")
```

### EventBridge Scheduler Setup

1. Go to **Amazon EventBridge → Schedules → Create schedule**
2. Set schedule type: **Recurring schedule (cron)**
3. Set target: **Lambda function** → `cost-optimization-EBS-snapshots`
4. Configure flexible time window: **5 minutes**
5. Set execution timezone: **Asia/Calcutta** (or your region)

---

## 📊 Real Execution Results

From the live CloudWatch log:
```
START RequestId: 56049ef4-3472-4ee4-b8c4-6bad011de778 Version: $LATEST

Deleted EBS snapshot snap-0a1920f1e46646372 as it was taken from a volume 
not attached to any running instance.

END RequestId: 56049ef4-3472-4ee4-b8c4-6bad011de778
REPORT Duration: 821.03 ms  Billed Duration: 822 ms  
Memory Size: 128 MB  Max Memory Used: 101 MB
```

---

## 💡 Why This Matters

EBS snapshots are one of the **most common hidden cost drivers** in AWS accounts. Developers create snapshots for backups, forget to clean them up, and they silently accumulate charges. This project automates the cleanup:

- No manual monitoring needed
- Runs on a defined schedule
- Logs every deletion for accountability
- Follows AWS Well-Architected Framework — Cost Optimization pillar

---

## 📌 Roadmap

- [ ] Add SNS notification on each deletion (email alert)
- [ ] Add dry-run mode (list snapshots without deleting)
- [ ] Extend to clean up unused Elastic IPs and unattached EBS volumes
- [ ] Terraform-based deployment

---

## 👩‍💻 Author

**Charmi [Last Name]**
B.Sc. (CA&IT) Honours — Ganpat University, Gujarat

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue?logo=linkedin)](https://linkedin.com/in/your-profile)
[![GitHub](https://img.shields.io/badge/GitHub-Follow-black?logo=github)](https://github.com/Charmi067)