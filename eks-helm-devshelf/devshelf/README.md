# DevShelf AWS EKS Migration (Production-Style Kubernetes Deployment)

## Project Overview

After successfully deploying DevShelf on a local Kind Kubernetes cluster, the next phase was migrating the complete application to Amazon Elastic Kubernetes Service (EKS).

The objective was to transform a locally running Kubernetes application into a cloud-native production-style architecture using:

- Amazon EKS
- Amazon ECR
- Helm
- Amazon EBS CSI Driver
- IAM Roles for Service Accounts (IRSA)
- AWS Load Balancer Controller
- Application Load Balancer (ALB)
- Dynamic Persistent Storage

---

# Local Kubernetes vs AWS EKS

| Local Kind Cluster | AWS EKS |
|--------------------|---------|
| Docker Images from Docker Hub | Images stored in Amazon ECR |
| NGINX Ingress Controller | AWS Application Load Balancer |
| Local PVC Storage | Amazon EBS gp3 Volumes |
| Docker Desktop Nodes | EC2 Worker Nodes |
| Manual Local Networking | AWS VPC Networking |

---

# Final Production Architecture

```
                           Internet
                               |
                               |
                  AWS Application Load Balancer
                               |
                               |
                       Kubernetes Ingress
                               |
              ----------------------------------
              |                                |
       Frontend Service                 Backend Service
          ClusterIP                       ClusterIP
              |                                |
        React + Nginx Pods               FastAPI Pods
                                               |
                                         Redis Cache
                                               |
                                          PostgreSQL
                                               |
                                              PVC
                                               |
                                          StorageClass
                                               |
                                          EBS CSI Driver
                                               |
                                          AWS EBS gp3
```

---

# Prerequisites

Installed:

- AWS CLI
- kubectl
- eksctl
- Helm
- Docker

Verification commands:

```bash
aws --version

kubectl version --client

eksctl version

helm version

docker --version
```

---

# AWS Authentication

Configured AWS credentials and verified access.

Command:

```bash
aws sts get-caller-identity
```

Example output:

```
Account ID
IAM User ARN
```

---

# EKS Cluster Creation

Created an EKS cluster using `eksctl`.

Command:

```bash
eksctl create cluster \
--name devshelf-eks \
--region us-east-1 \
--nodes 2
```

Verification:

```bash
kubectl get nodes
```

Expected:

```
ip-xxx.ec2.internal    Ready
ip-xxx.ec2.internal    Ready
```

---

# Amazon ECR Migration

## Created ECR repositories

- devshelf-backend
- devshelf-frontend


## Login to ECR

```bash
aws ecr get-login-password \
--region us-east-1 |
docker login \
--username AWS \
--password-stdin ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com
```

---

## Pull Docker Hub Images

```bash
docker pull smplynithin/devshelf-backend:latest

docker pull smplynithin/devshelf-frontend:latest
```

---

## Tag ECR Images

Backend:

```bash
docker tag smplynithin/devshelf-backend:latest \
ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/devshelf-backend:latest
```

Frontend:

```bash
docker tag smplynithin/devshelf-frontend:latest \
ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/devshelf-frontend:latest
```

---

## Push Images

```bash
docker push ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/devshelf-backend:latest

docker push ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/devshelf-frontend:latest
```

---

# Helm Chart Migration

Modified Helm values to use ECR repositories.

Example:

```yaml
backend:
  image:
    repository: ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/devshelf-backend
    tag: latest
```

Validated templates:

```bash
helm upgrade --install devshelf ./devshelf \
-n devshelf-prod \
-f secrets-values.yaml \
--dry-run --debug
```

---

# Amazon EBS CSI Driver Installation

## Purpose

Allows Kubernetes PVCs to dynamically provision Amazon EBS volumes.

Architecture:

```
PostgreSQL PVC
      |
StorageClass
      |
EBS CSI Driver
      |
AWS EBS Volume
```

---

## Enable OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
--cluster devshelf-eks \
--region us-east-1 \
--approve
```

---

## Create IRSA for EBS CSI Driver

Created IAM Role with:

```
AmazonEBSCSIDriverPolicy
```

Verified:

```bash
kubectl get sa ebs-csi-controller-sa \
-n kube-system -o yaml | grep role-arn
```

---

# StorageClass Creation

Created:

```yaml
name: ebs-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
```

Used:

```yaml
volumeBindingMode: WaitForFirstConsumer
```

to ensure EBS volumes are created in the same Availability Zone as the selected worker node.

---

# AWS Load Balancer Controller

## Purpose

Converts Kubernetes Ingress resources into AWS Application Load Balancers.

Architecture:

```
Ingress
   |
ALB Controller
   |
AWS API
   |
Application Load Balancer
```

---

## Created IAM Policy

Created:

```
AWSLoadBalancerControllerIAMPolicy
```

Attached it to a ServiceAccount using IRSA.

Verification:

```bash
kubectl get serviceaccount aws-load-balancer-controller \
-n kube-system -o yaml | grep role-arn
```

---

## Install Controller

```bash
helm install aws-load-balancer-controller \
eks/aws-load-balancer-controller \
-n kube-system \
--set clusterName=devshelf-eks
```

Verification:

```bash
kubectl get deployment \
aws-load-balancer-controller \
-n kube-system
```

---

# Ingress Migration

## Local Kind

```
NGINX Ingress Controller
```

## EKS

```
AWS ALB Ingress
```

Updated Ingress:

```yaml
annotations:
  alb.ingress.kubernetes.io/scheme: internet-facing
  alb.ingress.kubernetes.io/target-type: ip

ingressClassName: alb
```

---

# Challenges Faced and Resolution

---

## Challenge 1: Helm Value Structure Error

### Issue

Helm failed with:

```
nil pointer evaluating interface {}.repository
```

### Root Cause

The `values.yaml` image structure did not match the Helm templates.

### Resolution

Updated values to match:

```yaml
image:
  repository:
  tag:
```

---

## Challenge 2: ServiceMonitor CRD Missing

### Issue

Helm failed:

```
no matches for kind ServiceMonitor
```

### Root Cause

Prometheus Operator CRDs were not installed in EKS.

### Resolution

Removed the ServiceMonitor from the application Helm chart because monitoring infrastructure should be managed separately.

---

## Challenge 3: StorageClass Ownership Conflict

### Issue

Helm failed because `ebs-sc` already existed.

### Root Cause

StorageClass was created manually but also included in Helm.

### Resolution

Removed StorageClass from the application chart and treated it as cluster infrastructure.

---

## Challenge 4: EBS CSI Driver Permission Issue

### Issue

EBS CSI controller failed with:

```
UnauthorizedOperation
```

### Root Cause

The controller lacked IAM permissions.

### Resolution

Configured:

- OIDC provider
- IAM Role
- IRSA
- AmazonEBSCSIDriverPolicy

Restarted the controller to pick up the updated ServiceAccount.

---

## Challenge 5: PostgreSQL CrashLoopBackOff

### Issue

PostgreSQL failed:

```
directory exists but is not empty
lost+found directory
```

### Root Cause

New EBS volumes contain a Linux filesystem directory called `lost+found`.

### Resolution

Configured:

```yaml
PGDATA=/var/lib/postgresql/data/pgdata
```

so PostgreSQL initializes in a subdirectory.

---

## Challenge 6: ALB Not Created

### Issue

Ingress remained in pending state.

### Root Cause

AWS Load Balancer Controller was not installed and configured.

### Resolution

Installed ALB Controller with:

- IAM Policy
- IRSA
- Helm

---

# Application Deployment

Deploy DevShelf:

```bash
helm upgrade --install devshelf \
./devshelf \
-n devshelf-prod \
-f secrets-values.yaml \
--create-namespace
```

Verification:

```bash
kubectl get pods -n devshelf-prod

kubectl get pvc -n devshelf-prod

kubectl get ingress -n devshelf-prod
```

---

# Final Verification

Successful deployment:

- All Pods Running
- PostgreSQL PVC Bound
- AWS EBS Volume Created
- ALB DNS Generated
- Frontend Accessible
- Backend API Accessible

---

# Important EKS Learnings

This migration provided hands-on experience with:

- Amazon EKS cluster management
- Amazon ECR image management
- Helm-based deployments
- Kubernetes RBAC
- Stateful applications on EKS
- Dynamic EBS provisioning
- OIDC and IRSA
- AWS Load Balancer Controller
- ALB Ingress
- Production troubleshooting
- Infrastructure vs Application separation

---

# Cleanup (Cost Optimization)

Delete EKS cluster:

```bash
eksctl delete cluster \
--name devshelf-eks \
--region us-east-1
```

Verify removal of:

- EC2 worker nodes
- Load Balancers
- EBS volumes
- CloudFormation stacks

---

# Conclusion

DevShelf was successfully migrated from a local Kind cluster to a production-style Amazon EKS environment.

The project demonstrates not only Kubernetes deployment skills but also real-world cloud troubleshooting involving IAM permissions, storage provisioning, Helm architecture, and AWS networking.
