# EKS 2048 Game Deployment Guide

This guide explains how to deploy the 2048 game on **Amazon EKS (Elastic Kubernetes Service)** using **Fargate** and an **ALB (Application Load Balancer)**.

---

## What is EKS?

Amazon EKS is a managed Kubernetes service that makes it easy to run Kubernetes on AWS without needing to install and operate your own Kubernetes control plane or nodes.

---

## Prerequisites

Before starting, make sure you have the following installed:

* **kubectl** – A command-line tool for working with Kubernetes clusters.
  [Installing or updating kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)

* **eksctl** – A command-line tool for working with EKS clusters that automates many individual tasks.
  [Installing or updating eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)

* **AWS CLI** – A command-line tool for working with AWS services, including Amazon EKS.
  [Installing, updating, and uninstalling the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)
  After installing, configure it using:

  ```bash
  aws configure
  ```

---

## Install EKS Cluster

### Using Fargate

Create a cluster with Fargate:

```bash
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```

### Delete the cluster (if needed)

```bash
eksctl delete cluster --name demo-cluster --region us-east-1
```

---

## 2048 Game App Deployment

### 1. Create Fargate profile

```bash
eksctl create fargateprofile \
    --cluster demo-cluster \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048
```

### 2. Deploy the game (deployment, service, and ingress)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```
![Screenshot 2023-08-03 at 7 57 15 PM](https://github.com/iam-veeramalla/aws-devops-zero-to-hero/assets/43399466/93b06a9f-67f9-404f-b0ad-18e3095b7353)

### 3. Check IAM OIDC provider

Verify if an OIDC provider exists:

```bash
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```

If not, associate it:

```bash
eksctl utils associate-iam-oidc-provider --cluster demo-cluster --approve
```

---

## Setup ALB Controller

### 1. Download IAM policy

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

### 2. Create IAM Policy

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

### 3. Create IAM Role

```bash
eksctl create iamserviceaccount \
  --cluster=demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

### 4. Deploy ALB Controller using Helm

Add the Helm repository:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```

Install the controller:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<your-vpc-id>
```

Verify deployment:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

## Verify the Deployment

After deploying the 2048 app and ALB controller, you can check the status of your cluster and application using the following commands:

### 1. Check all nodes

```bash
kubectl get nodes
```

### 2. Check all namespaces

```bash
kubectl get namespaces
```

### 3. Check all pods in the game namespace

```bash
kubectl get pods -n game-2048
```

### 4. Check deployments in the namespace

```bash
kubectl get deployments -n game-2048
```

### 5. Check services

```bash
kubectl get svc -n game-2048
```

### 6. Check ingress and app address

```bash
kubectl get ingress -n game-2048 -w
```

Example output:

```
NAME           CLASS   HOSTS   ADDRESS                                                              PORTS   AGE
ingress-2048   alb     *       k8s-game2048-ingress2-bcac0b5b37-1314048833.eu-west-1.elb.amazonaws.com   80      86m
```

* The **ADDRESS** column shows the URL where your 2048 app is accessible.
* Open this address in a browser to play the game.

---

## Screenshot


