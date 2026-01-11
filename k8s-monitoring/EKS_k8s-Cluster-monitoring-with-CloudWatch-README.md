
# EKS Monitoring with Amazon CloudWatch

This repository explains step-by-step how to monitor an Amazon EKS cluster using the Amazon CloudWatch Observability add-on.

---
## Step 0: Prerequisites

Ensure the following are installed and configured:

- AWS Account
- AWS CLI (`aws configure`)
- eksctl
- IAM permissions for:
  - EKS
  - EC2
  - IAM
  - CloudWatch

---

## Install AWS CLI
```bash
sudo apt update
curl -o awscliv2.zip https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
```

## Configure AWS CLI:
```bash
aws configure
```

Enter your Access Key ID, Secret Access Key, region (e.g., `us-east-1`), and output format when prompted.

---

## NOTE: IF you don't wish to use SECRETE_KEY & SECRETE_ACCESS_KEYS you can use the IAM ROle with ADMINISTRATOR policy and attach that policy to EC2 Instance.

---

## Install kubectl
```bash
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
```

## Verify installation:
```bash
kubectl version --short --client
```

---

## Install eksctl
```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

## Verify installation:
```bash
eksctl version
```

## Step 1: Create EKS Cluster

Create an Amazon EKS cluster with managed node groups.
---
```bash
eksctl create cluster \
  --name ekswithshaik \
  --version 1.33 \
  --region ap-south-1 \
  --zones ap-south-1a,ap-south-1b \
  --nodegroup-name ng-default \
  --node-type t3.small \
  --nodes 2 \
  --node-ami-family AmazonLinux2023 \
  --managed
````

---

## Step 2: Associate IAM OIDC Provider

This step enables IAM Roles for Service Accounts (IRSA).
---
```bash
eksctl utils associate-iam-oidc-provider \
  --region ap-south-1 \
  --cluster ekswithshaik \
  --approve
```

---

## Step 3: Attach CloudWatch Policy to Worker Node Role

Attach the AWS managed policy below to the EKS worker node IAM role:

```
CloudWatchAgentServerPolicy
```

This policy allows worker nodes to send logs and metrics to CloudWatch.

---

mv ~/.kube/config ~/.kube/config.bak

## Update the kubeconfig:
```bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name ekswithshaik
```

## Step 4: Install CloudWatch Observability Add-on

Install the official Amazon CloudWatch Observability add-on.

```bash
aws eks create-addon \
  --addon-name amazon-cloudwatch-observability \
  --cluster-name ekswithshaik \
  --region ap-south-1
```

This installs:

* CloudWatch Agent
* Fluent Bit
* Container Insights

---

## Step 5: Verify Add-on Status

Check whether the add-on is active.

```bash
aws eks describe-addon \
  --addon-name amazon-cloudwatch-observability \
  --cluster-name ekswithshaik \
  --region ap-south-1
```

Expected output status:

```
ACTIVE
```

---

## Step 6: View Metrics and Logs

### Metrics

* AWS Console → CloudWatch → Container Insights

### Logs

* CloudWatch → Log Groups
* `/aws/containerinsights/ekswithshaik/application`

---

## Step 7: Cleanup Resources

Delete the EKS cluster to avoid unnecessary charges.

```bash
eksctl delete cluster \
  --name ekswithshaik \
  --region ap-south-1
```

---

## Benefits

* Native AWS monitoring solution
* No manual agent installation
* Scales automatically
* Production-ready observability

---

## Author

**Moula Shaik**
**Cloud&DevOps Consultant**
**DevOps | AWS | Kubernetes**

```
