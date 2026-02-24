# 🚀 EKS 2048 Game Deployment & Cleanup – Command Reference (Markdown)
---

## 📌 Overview
This document explains **all commands** used to:
- Create an **EKS cluster with Fargate**
- Deploy the **2048 demo game** using **AWS Load Balancer Controller**
- Expose the app using **ALB Ingress**
- Clean up **all resources** to avoid billing

---

## 🧱 1. Create EKS Cluster with Fargate

```bash
eksctl create cluster --name my-cluster-1187 --region ap-south-1 --fargate
```

### 🔍 Explanation
- Creates an **EKS cluster** named `my-cluster-1187`
- Uses **Mumbai region (ap-south-1)**
- Enables **Fargate**, so no EC2 worker nodes are needed

📌 **Cost Note:** EKS control plane costs `$0.10/hour`

---

## 🔐 2. Associate IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider --cluster my-cluster-1187 --approve
```

### 🔍 Explanation
- Enables **OIDC** for the cluster
- Required for **IAM Roles for Service Accounts (IRSA)**
- Mandatory for AWS Load Balancer Controller

---

## 📥 3. Download IAM Policy for ALB Controller

```powershell
Invoke-WebRequest \
 -Uri https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json \
 -OutFile iam_policy.json
```

### 🔍 Explanation
- Downloads the **official IAM policy** required by ALB Controller

---

## 🔐 4. Create IAM Policy

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

### 🔍 Explanation
- Creates an IAM policy with required permissions to manage ALBs
- If policy already exists, AWS will return `EntityAlreadyExists`

---

## 👤 5. Create IAM Service Account

```bash
eksctl create iamserviceaccount \
  --cluster my-cluster-1187 \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

### 🔍 Explanation
- Creates a **Kubernetes service account**
- Attaches IAM role using **IRSA**
- Allows ALB controller to call AWS APIs securely

---

## 📦 6. Install AWS Load Balancer Controller (Helm)

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster-1187 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-south-1 \
  --set vpcId=<YOUR_VPC_ID>
```

### 🔍 Explanation
- Installs the controller that automatically creates **ALBs** for Kubernetes Ingress

---

## 🚀 7. Create Fargate Profile for Application

```bash
eksctl create fargateprofile \
  --cluster my-cluster-1187 \
  --region ap-south-1 \
  --name alb-sample-app \
  --namespace game-2048
```

### 🔍 Explanation
- Allows pods in `game-2048` namespace to run on **Fargate**

---

## 🎮 8. Deploy 2048 Game Application

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

### 🔍 Explanation
- Creates:
  - Namespace `game-2048`
  - Deployment
  - Service
  - Ingress (ALB)

---

## 🔍 9. Verify Application Resources

```bash
kubectl get pods -n game-2048
kubectl get pods -n game-2048 -w
kubectl get svc -n game-2048
kubectl get ingress -n game-2048
```

### 🔍 Explanation
- Confirms pods are running
- Service exposes pods internally
- Ingress exposes app publicly via ALB

---

## 🧹 10. DELETE APPLICATION (STOP BILLING)

### 🔴 Delete Ingress (Deletes ALB)

```bash
kubectl delete ingress ingress-2048 -n game-2048
```

### 🔴 Delete Service

```bash
kubectl delete svc service-2048 -n game-2048
```

### 🔴 Delete Deployment

```bash
kubectl delete deployment deployment-2048 -n game-2048
```

### 🔴 Delete Namespace

```bash
kubectl delete ns game-2048
```

---

## 🧹 11. Remove AWS Load Balancer Controller

```bash
helm uninstall aws-load-balancer-controller -n kube-system
```

```bash
eksctl delete iamserviceaccount \
  --cluster my-cluster-1187 \
  --namespace kube-system \
  --name aws-load-balancer-controller
```

---

## 🧹 12. Delete Fargate Profiles

```bash
eksctl get fargateprofile --cluster my-cluster-1187
```

```bash
eksctl delete fargateprofile --cluster my-cluster-1187 --name alb-sample-app
eksctl delete fargateprofile --cluster my-cluster-1187 --name fp-default
```

---

## 🧨 13. Delete EKS Cluster (FINAL STEP)

```bash
eksctl delete cluster --name my-cluster-1187 --region ap-south-1
```

```bash
eksctl get cluster
```

### 🔍 Explanation
- Completely removes EKS
- Stops **all AWS billing related to this setup**

---

## ✅ Final Zero-Billing Checklist

✔ EKS cluster deleted
✔ ALB deleted
✔ NAT Gateway deleted
✔ No EC2 instances
✔ No running Fargate pods

---

## 🎯 Interview-Ready Summary

> “I deployed a 2048 demo application on AWS EKS using Fargate and ALB Ingress, secured it with IAM Roles for Service Accounts, and safely cleaned up all resources to avoid billing.”

---

✍️ **Author:** Hariom Lokhandkar

