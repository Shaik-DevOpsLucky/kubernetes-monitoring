# Grafana CloudWatch Monitoring Setup for AWS RDS (EKS + Prometheus Stack)

## Overview

This guide explains how to monitor **AWS RDS PostgreSQL metrics in Grafana** using **CloudWatch as a data source** when Grafana is running inside an **EKS Kubernetes cluster**.

Architecture:

```
Amazon RDS
     │
     ▼
Amazon CloudWatch
     │
     ▼
Grafana (running in EKS)
     │
     ▼
Dashboards & Alerts
```

This setup uses **IAM Roles for Service Accounts (IRSA)** so that Grafana securely accesses CloudWatch without using AWS access keys.

---

# Prerequisites

Make sure the following are already configured:

* AWS Account
* EKS Cluster
* RDS PostgreSQL instance
* kubectl configured
* eksctl installed
* Grafana + Prometheus installed via kube-prom-stack
* IAM OIDC provider enabled for EKS

Verify cluster access:

```bash
kubectl get nodes
```

Verify Grafana deployment:

```bash
kubectl get pods -n monitoring
```

Example output:

```
kube-prom-stack-grafana-xxxx
```

---

# Step 1 — Create IAM Policy for CloudWatch Access

Create a policy that allows Grafana to read CloudWatch metrics and logs.

Go to AWS Console → IAM → Policies → Create Policy.

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

Save the policy as:

```
GrafanaCloudWatchPolicy
```

---

# Step 2 — Verify Grafana ServiceAccount

Check which ServiceAccount Grafana is using.

```bash
kubectl get deployment kube-prom-stack-grafana -n monitoring -o yaml | grep serviceAccountName
```

Expected output:

```
serviceAccountName: kuincbe-prom-stack-grafana
```

So the ServiceAccount is:

```
kube-prom-stack-grafana
```

Namespace:

```
monitoring
```

---
# NOTE: If service account not there, create it manually.
```
kubectl create serviceaccount kube-prom-stack-grafana -n monitoring
```
# Verify:
```
kubectl get sa -n monitoring
```


# Step 3 — Create IAM Role for Grafana (IRSA)

Attach the IAM policy to the Grafana ServiceAccount.

Run:

```bash
eksctl create iamserviceaccount \
  --name kube-prom-stack-grafana \
  --namespace monitoring \
  --cluster Shaik-Ecom-cluster \
  --region us-east-1 \
  --attach-policy-arn arn:aws:iam::<ACCOUNT-ID>:policy/GrafanaCloudWatchPolicy \
  --override-existing-serviceaccounts \
  --approve
```

Explanation:

| Parameter                         | Description                  |
| --------------------------------- | ---------------------------- |
| name                              | Kubernetes ServiceAccount    |
| namespace                         | monitoring namespace         |
| cluster                           | EKS cluster name             |
| region                            | AWS region                   |
| override-existing-serviceaccounts | attaches role to existing SA |

---

# Step 4 — Verify IAM Role Annotation

Check the ServiceAccount:

```bash
kubectl get sa -n monitoring
kubectl describe sa kube-prom-stack-grafana -n monitoring
```

Expected annotation:

```
eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT-ID>:role/eksctl-Shaik-Ecom-cluster-addon-iamserviceaccount-Role
```

This confirms IRSA is working.

---
# IF SA details not updated by default, you need to update your deployment with SA details: (It has added by default)

# 🔥 Next Step (Important)

After creating the service account, you must **attach it to Grafana deployment**.

Edit Grafana:

```bash
kubectl edit deployment kube-prom-stack-grafana -n monitoring
```

Add:

```yaml
serviceAccountName: kube-prom-stack-grafana-sa
```

Then restart:

```bash
kubectl rollout restart deployment kube-prom-stack-grafana -n monitoring
```

---

✅ After this your **Grafana running in Kubernetes on Amazon EKS will securely read metrics from Amazon CloudWatch without access keys.

---

# Step 5 — Restart Grafana

Restart the deployment so the new role is used.

```bash
kubectl rollout restart deployment kube-prom-stack-grafana -n monitoring
```

Verify pod restart:

```bash
kubectl get pods -n monitoring
```

---

# Step 6 — Add CloudWatch Data Source in Grafana

Open Grafana UI.

Navigate to:

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

Default Region:

```
us-east-1
```

Leave Access Key fields empty (IRSA will handle authentication).

Click:

```
Save & Test
```

Successful message:

```
Successfully queried the CloudWatch metrics API
Successfully queried the CloudWatch logs API
```

---

# Step 7 — Query RDS Metrics

Create a dashboard.

Add panel → Select CloudWatch datasource.

Namespace:

```
AWS/RDS
```

Example metric query:

```
Metric: CPUUtilization
Dimension: DBInstanceIdentifier
Statistic: Average
Period: 5 minutes
```

---

# Step 8 — Recommended RDS Metrics

Add panels for these metrics.

| Metric              | Description             |
| ------------------- | ----------------------- |
| CPUUtilization      | Database CPU usage      |
| DatabaseConnections | Active DB connections   |
| FreeableMemory      | Available memory        |
| FreeStorageSpace    | Disk storage remaining  |
| ReadLatency         | Read operation latency  |
| WriteLatency        | Write operation latency |
| DiskQueueDepth      | Storage IO backlog      |

These metrics give full database health visibility.

---

# Step 9 — Create Dashboard Variables (Multiple DBs)

To monitor multiple RDS instances dynamically:

Dashboard → Settings → Variables → Add variable.

Configuration:

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

This allows switching between DB instances.

---

# Step 10 — Create Grafana Alerts

Navigate:

```
Grafana → Alerting → Alert Rules
```

Recommended alerts:

### High CPU

Condition:

```
CPUUtilization > 80%
Duration: 5 minutes
```

### Low Storage

```
FreeStorageSpace < 10GB
```

### High Database Connections

```
DatabaseConnections > 80%
```

Alerts can send notifications to:

* Slack
* Email
* PagerDuty
* Webhooks

---

# Step 11 — CloudWatch Logs Query (Optional)

If RDS logs are exported to CloudWatch, you can visualize logs.

Example query:

```
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20
```

Useful for:

* PostgreSQL errors
* Slow queries
* Authentication failures

---

# Step 12 — Backup Grafana Dashboards

Since Grafana runs in Kubernetes, dashboards should be backed up.

Export dashboards:

```
Dashboard → Settings → JSON Model
```

Save JSON into Git repository.

---

# Final Architecture

```
Amazon RDS (PostgreSQL)
        │
        ▼
Amazon CloudWatch
        │
        ▼
Grafana (Running in EKS)
        │
        ▼
Dashboards + Alerts
```

---

# Best Practices

1. Always use **IRSA instead of AWS Access Keys**
2. Monitor **CPU, Memory, Storage, Connections**
3. Configure **alerts for production environments**
4. Store dashboards in **Git**
5. Use dashboard **variables for multiple RDS instances**

---

# Future Improvements

Advanced monitoring architecture:

```
CloudWatch
     │
     ▼
YACE Exporter
     │
     ▼
Prometheus
     │
     ▼
Grafana
```

Benefits:

* Long metric retention
* Faster queries
* Better Prometheus alerting
* Central monitoring for AWS services

---

# Useful Commands

Check Grafana pod:

```
kubectl get pods -n monitoring
```

Restart Grafana:

```
kubectl rollout restart deployment kube-prom-stack-grafana -n monitoring
```

Check ServiceAccount:

```
kubectl describe sa kube-prom-stack-grafana -n monitoring
```

---

# Result

After completing this setup you will have:

✔ Secure authentication using IRSA
✔ CloudWatch integrated with Grafana
✔ RDS infrastructure monitoring
✔ Dashboard visualization
✔ Alerting for database health

---

# Prepared by:
*Shaik Moulali*
