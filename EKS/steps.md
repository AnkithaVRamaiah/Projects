
# Project Name

# Serverless Kubernetes Application Deployment on AWS EKS using Fargate and ALB

---

## Problem Statement

Running Kubernetes applications requires managing infrastructure like servers (EC2 nodes), load balancers, and security permissions, which can be complex, costly, and error-prone. Traditional Kubernetes setups need you to manage nodes, expose apps securely, and control access permissions carefully.

---

## What This Project Solves

* **No need to manage servers** — uses AWS Fargate (serverless pods).
* **Easy app exposure to the internet** — uses AWS Application Load Balancer (ALB) controlled by Kubernetes Ingress.
* **Secure access control** — uses OIDC and IAM Roles for Service Accounts (IRSA) to give pods minimal and specific AWS permissions safely.
* **Automated and simplified cluster setup** — uses eksctl, kubectl, AWS CLI, and Helm tools.

---

## Step-by-Step Guide with Explanation

### Step 1: Install Tools — `kubectl`, `eksctl`, and `aws cli`

* **Why?**

  * `aws cli` connects your computer with your AWS account.
  * `eksctl` simplifies creating and managing EKS Kubernetes clusters on AWS.
  * `kubectl` is the Kubernetes command-line tool to control your cluster and apps.
* **Interview Tip:**
  We need these three because AWS CLI talks to AWS, eksctl manages the cluster creation, and kubectl interacts with the Kubernetes workloads.

---

### Step 2: Configure AWS CLI with `aws configure`

* **Why?**
  This sets up your AWS credentials (Access Key and Secret) and default region. Without this, none of your AWS commands will authenticate and work.
* **Interview Tip:**
  If skipped, commands fail because AWS doesn’t know who you are.

---

### Step 3: Create EKS Cluster with Fargate using `eksctl`

* Command:

  ```bash
  eksctl create cluster --name demo-cluster --region us-east-1 --fargate
  aws eks --region us-east-1 update-kubeconfig --name demo-cluster
  ```
* **Why?**
  Creates a Kubernetes cluster on AWS, but instead of managing EC2 nodes, it uses Fargate to run pods serverlessly. This removes the complexity of node management.
* **Interview Tip:**
  Fargate provides serverless compute, auto scales, and you only pay for what you use.

---

### Step 4: Create a Fargate Profile for Namespace

* Command:

  ```bash
  eksctl create fargateprofile --cluster demo-cluster --region us-east-1 --name alb-sample-app --namespace game-2048
  ```
* **Why?**
  This tells EKS to run all pods inside the `game-2048` namespace on Fargate.
* **Interview Tip:**
  A Fargate profile links Kubernetes namespaces to Fargate, controlling which pods run serverless.

---

### Step 5: Deploy Your Sample App (Pods, Service, Ingress)

* Command:

  ```bash
  kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.13.2/docs/examples/2048/2048_full.yaml
  ```
* **Why?**
  This deploys the app, creates a Kubernetes service to expose it internally, and an Ingress resource to expose it externally using an ALB.
* **Interview Tip:**
  Ingress routes incoming traffic via ALB to your service and pods inside the cluster.

---

### Step 6: Associate IAM OIDC Provider with EKS Cluster

* Command:

  ```bash
  eksctl utils associate-iam-oidc-provider --cluster demo-cluster --approve
  ```
* **Why?**
  This step allows AWS to trust Kubernetes service accounts via OpenID Connect (OIDC), enabling secure IAM role assignments to pods.
* **Interview Tip:**
  OIDC is a way to authenticate Kubernetes pods with AWS IAM securely.

---

### Step 7: Create IAM Policy and IAM Role for AWS Load Balancer Controller

* Commands:

  ```bash
  curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json

  aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

  eksctl create iamserviceaccount \
    --cluster=demo-cluster \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --role-name AmazonEKSLoadBalancerControllerRole \
    --attach-policy-arn=arn:aws:iam::<your-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
    --approve
  ```
* **Why?**
  The ALB controller needs AWS permissions to create and manage load balancers. We attach these permissions only to its Kubernetes service account for security (least privilege).
* **Interview Tip:**
  Using IRSA, only the ALB controller pod can manage AWS ALBs, improving security.

---

### Step 8: Install AWS Load Balancer Controller using Helm

* Commands:

  ```bash
  helm repo add eks https://aws.github.io/eks-charts
  helm repo update

  helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
    --set clusterName=demo-cluster \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller \
    --set region=us-east-1 \
    --set vpcId=<your-vpc-id>
  ```
* **Why?**
  The ALB controller watches for Ingress objects and creates AWS ALBs dynamically to expose your app. Helm makes installing and configuring this complex controller easier.
* **Interview Tip:**
  Helm is like a package manager for Kubernetes apps.

---

### Step 9: Access the Application via ALB DNS

* Command:

  ```bash
  kubectl get ingress -n game-2048
  ```
* Open the DNS address shown in a browser to access your app.
* **Why?**
  The ALB controller has created a load balancer that routes traffic from outside the internet to your Kubernetes service and pods running on Fargate.
* **Interview Tip:**
  Traffic flows: Browser → ALB → Ingress → Kubernetes Service → Pod → Your app.

---

# Summary for Interview

> "I built a fully managed Kubernetes app on AWS EKS using Fargate for serverless pods, exposing the app securely with an Application Load Balancer using Kubernetes Ingress. I configured OIDC and IAM Roles for Service Accounts (IRSA) to grant the app fine-grained AWS permissions securely. Tools like eksctl, kubectl, AWS CLI, and Helm automated and simplified the setup."

---

# Bonus: Project Diagram (Textual)

```
[User Browser]
       |
       v
[ALB (AWS Load Balancer)]
       |
       v
[Ingress Resource (Kubernetes)]
       |
       v
[Kubernetes Service]
       |
       v
[Pods (Running on Fargate)]
       |
       v
[IAM Role via IRSA] ← [OIDC Provider] ← [AWS IAM]
```

---


