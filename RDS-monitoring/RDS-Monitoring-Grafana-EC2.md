# Grafana CloudWatch Monitoring Setup for AWS RDS (Grafana on EC2)

## Overview

This guide explains how to monitor **AWS RDS PostgreSQL metrics in Grafana** using **Amazon CloudWatch** when **Grafana is installed on an AWS EC2 instance**.

Architecture:

```
Amazon RDS
     │
     ▼
Amazon CloudWatch
     │
     ▼
Grafana (Running on EC2)
     │
     ▼
Dashboards + Alerts
```

Authentication is handled using an **IAM Role attached to the EC2 instance**.

This avoids storing **AWS Access Keys** inside Grafana.

---

# Prerequisites

Ensure the following are available:

* AWS Account
* EC2 instance with Grafana installed
* RDS PostgreSQL instance
* IAM permissions to create roles and policies
* CloudWatch metrics enabled

Verify Grafana is running:

```
sudo systemctl status grafana-server
```

Open Grafana UI:

```
http://<EC2-PUBLIC-IP>:3000
```

Default login:

```
username: admin
password: admin
```

---

# Step 1 — Create IAM Policy for CloudWatch Access

Go to:

```
AWS Console → IAM → Policies → Create Policy
```

Policy JSON:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:ListMetrics",
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:GetMetricData"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams",
        "logs:GetLogEvents",
        "logs:StartQuery",
        "logs:StopQuery",
        "logs:GetQueryResults"
      ],
      "Resource": "*"
    }
  ]
}
```

Save policy as:

```
GrafanaCloudWatchPolicy
```

---

# Step 2 — Create IAM Role for EC2

Navigate to:

```
IAM → Roles → Create Role
```

Select:

```
Trusted Entity → AWS Service
```

Choose:

```
EC2
```

Attach policy:

```
GrafanaCloudWatchPolicy
```

Role Name:

```
Grafana-CloudWatch-Role
```

Create the role.

---

# Step 3 — Attach IAM Role to EC2 Instance

Navigate:

```
EC2 → Instances
```

Select your **Grafana EC2 instance**

Actions:

```
Security → Modify IAM Role
```

Attach role:

```
Grafana-CloudWatch-Role
```

Save changes.

---

# Step 4 — Verify IAM Role on EC2

SSH into EC2:

```
ssh ec2-user@<EC2-IP>
```

Test IAM role:

```
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

Expected output:

```
Grafana-CloudWatch-Role
```

This confirms EC2 can access AWS APIs.

---

# Step 5 — Add CloudWatch Datasource in Grafana

Open Grafana UI.

Navigate:

```
Connections → Data Sources → Add Data Source
```

Select:

```
CloudWatch
```

Configuration:

Authentication Provider:

```
AWS SDK Default
```

Region:

```
us-east-1
```

Leave Access Key fields empty because IAM Role will provide credentials.

Click:

```
Save & Test
```

Expected result:

```
Successfully queried the CloudWatch metrics API
Successfully queried the CloudWatch logs API
```

---

# Step 6 — Create RDS Dashboard

Create dashboard.

Add panel → Select datasource **CloudWatch**

Namespace:

```
AWS/RDS
```

Example query:

```
Metric: CPUUtilization
Dimension: DBInstanceIdentifier
Statistic: Average
Period: 5 minutes
```

---

# Step 7 — Recommended RDS Metrics

Add panels for:

| Metric              | Description        |
| ------------------- | ------------------ |
| CPUUtilization      | CPU usage          |
| DatabaseConnections | Active connections |
| FreeableMemory      | Memory available   |
| FreeStorageSpace    | Remaining storage  |
| ReadLatency         | Read latency       |
| WriteLatency        | Write latency      |
| DiskQueueDepth      | Storage IO backlog |

---

# Step 8 — Create Dashboard Variable (Multiple Databases)

Navigate:

```
Dashboard Settings → Variables
```

Add variable:

```
Name: dbinstance
Type: Query
Datasource: CloudWatch
Namespace: AWS/RDS
Dimension: DBInstanceIdentifier
```

Use in queries:

```
DBInstanceIdentifier = $dbinstance
```

This allows switching between multiple RDS instances.

---

# Step 9 — Grafana Alerting Setup

Navigate:

```
Grafana → Alerting → Alert Rules
```

Click:

```
New Alert Rule
```

---

## Example Alert — High CPU

Rule Name:

```
RDS High CPU
```

Query:

```
Datasource: CloudWatch
Namespace: AWS/RDS
Metric: CPUUtilization
Statistic: Average
Period: 5m
```

Condition:

```
WHEN avg() OF query(A) IS ABOVE 80
FOR 5 minutes
```

Meaning:

```
CPUUtilization > 80% for 5 minutes
```

Save the rule.

---

# Step 10 — Create Notification Contact Points

Navigate:

```
Alerting → Contact Points
```

Example Email:

```
Name: email-alert
Type: Email
Address: your-email@gmail.com
```

Example Slack:

```
Name: slack-alert
Type: Slack
Webhook URL: <SLACK_WEBHOOK>
```

Save.

---

# Step 11 — Create Notification Policy

Navigate:

```
Alerting → Notification Policies
```

Example:

```
Folder: General
Send To: email-alert
```

Now alerts will trigger notifications.

---

# Step 12 — Optional CloudWatch Logs Queries

If RDS logs are exported to CloudWatch:

Example query:

```
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20
```

Useful for:

* Database errors
* Slow queries
* Authentication failures

---

# Step 13 — Backup Grafana Dashboards

Export dashboards:

```
Dashboard → Settings → JSON Model
```

Store JSON files in Git.

---

# Useful Commands

Restart Grafana:

```
sudo systemctl restart grafana-server
```

Check Grafana status:

```
sudo systemctl status grafana-server
```

Check EC2 IAM role:

```
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

---

# Final Architecture

```
Amazon RDS
     │
     ▼
Amazon CloudWatch
     │
     ▼
Grafana (EC2)
     │
     ▼
Dashboards + Alerts
```

---

# Best Practices

* Always use **IAM Role instead of Access Keys**
* Monitor **CPU, Memory, Storage**
* Configure **alerts for production**
* Store dashboards in **Git**
* Use **dashboard variables**

---

# Result

After completing this setup you will have:

✔ Secure authentication using IAM Role
✔ CloudWatch integrated with Grafana
✔ RDS monitoring dashboards
✔ Alerting configured
✔ Slack / Email notifications

# Prepared by:
*Shaik Moulali*
