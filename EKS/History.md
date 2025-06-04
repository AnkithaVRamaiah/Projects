$ eksctl create cluster --name demo-cluster --region us-east-1 --fargate
[ℹ] eksctl version 0.191.0
[ℹ] using region us-east-1
[ℹ] setting availability zones to [us-east-1f us-east-1a]
[ℹ] subnets for us-east-1f - public:192.168.0.0/19 private:192.168.64.0/19
[ℹ] subnets for us-east-1a - public:192.168.32.0/19 private:192.168.96.0/19
[ℹ] using Kubernetes version 1.30
[ℹ] creating EKS cluster "demo-cluster" in "us-east-1" region with Fargate profile
[ℹ] Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "demo-cluster"
[ℹ] CloudWatch logging will not be enabled for cluster "demo-cluster"
[ℹ] default addons kube-proxy, coredns, vpc-cni were not specified, will install them as EKS addons
[ℹ] 2 sequential tasks: { create cluster control plane "demo-cluster", wait for control plane to become ready, create fargate profiles, create addons }
[ℹ] building cluster stack "eksctl-demo-cluster-cluster"
[ℹ] deploying stack "eksctl-demo-cluster-cluster"
[ℹ] waiting for CloudFormation stack "eksctl-demo-cluster-cluster" (multiple times)
[ℹ] creating addon (multiple times)
[!] recommended policies were found for "vpc-cni" addon, but since OIDC is disabled on the cluster, eksctl cannot configure the requested permissions
[ℹ] creating Fargate profile "fp-default" on EKS cluster "demo-cluster"
[ℹ] created Fargate profile "fp-default" on EKS cluster "demo-cluster"
[ℹ] "coredns" is now schedulable onto Fargate
[ℹ] "coredns" pods are now scheduled onto Fargate
[ℹ] waiting for the control plane to become ready
[✔] saved kubeconfig
[✔] all EKS cluster resources for "demo-cluster" have been created
[✔] created 0 nodegroup(s) in cluster "demo-cluster"
[✔] created 0 managed nodegroup(s) in cluster "demo-cluster"
[ℹ] kubectl command should work with your kubeconfig, try 'kubectl get nodes'
[✔] EKS cluster "demo-cluster" in "us-east-1" region is ready

$ aws eks --region us-east-1 update-kubeconfig --name demo-cluster
Added new context for cluster/demo-cluster to kubeconfig

$ kubectl get svc
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.100.0.1    <none>        443/TCP   10m

$ kubectl get pods -n game-2048
No resources found in game-2048 namespace.

$ eksctl create fargateprofile --cluster demo-cluster --region us-east-1 --name alb-sample-app --namespace game-2048
[ℹ] creating Fargate profile "alb-sample-app" on EKS cluster "demo-cluster"
[ℹ] created Fargate profile "alb-sample-app" on EKS cluster "demo-cluster"

$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
namespace/game-2048 created
deployment.apps/deployment-2048 created
service/service-2048 created
ingress.networking.k8s.io/ingress-2048 created

$ eksctl utils associate-iam-oidc-provider --cluster demo-cluster --approve
[ℹ] created IAM Open ID Connect provider for cluster "demo-cluster" in "us-east-1"

$ kubectl get pods -n game-2048
NAME                               READY   STATUS    RESTARTS   AGE
deployment-2048-85f8c7d69-xxxxx   1/1     Running   0          86s
(deployment-2048 replicated pods)

$ curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
# Downloaded iam_policy.json (removed download stats)

$ aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json
# Command ran (no output shown)

$ eksctl create iamserviceaccount --cluster=demo-cluster --namespace=kube-system --name=aws-load-balancer-controller --role-name AmazonEKSLoadBalancerControllerRole --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy --approve
[ℹ] 1 iamserviceaccount (kube-system/aws-load-balancer-controller) was included
[ℹ] created serviceaccount "kube-system/aws-load-balancer-controller"

$ helm repo add eks https://aws.github.io/eks-charts
"eks" already exists with the same configuration, skipping

$ helm repo update eks
Successfully got an update from the "eks" chart repository

$ helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=demo-cluster --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller --set region=us-east-1 --set vpcId=<VPC_ID>
NAME: aws-load-balancer-controller
LAST DEPLOYED: ...
NAMESPACE: kube-system
STATUS: deployed

$ kubectl get deployment -n kube-system aws-load-balancer-controller
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller  0/2     2            0           30s

$ kubectl get pods -n game-2048
NAME                               READY   STATUS    RESTARTS   AGE
deployment-2048-85f8c7d69-xxxxx   1/1     Running   0          25m
...

$ kubectl get deployment -n kube-system aws-load-balancer-controller
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller  2/2     2            2           96s

$ kubectl get ing -n game-2048
NAME          CLASS   HOSTS   ADDRESS                                                       PORTS   AGE
ingress-2048  alb     *       k8s-game2048-ingress2-xxxx.us-east-1.elb.amazonaws.com        80      28m

$ eksctl delete cluster --name demo-cluster --region us-east-1
[ℹ] deleting EKS cluster "demo-cluster"
[ℹ] deleting Fargate profile "alb-sample-app"
[ℹ] deleted Fargate profile "alb-sample-app"
[ℹ] deleting Fargate profile "fp-default"
[ℹ] deleted Fargate profile "fp-default"
[ℹ] deleted 2 Fargate profile(s)
[ℹ] cleaning up AWS load balancers created by Kubernetes objects of Kind Service or Ingress
[!] error when checking existence of load balancer k8s-game2048-ingress2-xxxxx: operation error Elastic Load Balancing v2: DescribeLoadBalancers, https response error StatusCode: 0, RequestID: , canceled, context deadline exceeded
Error: cannot delete orphan ELB Security Groups: cannot describe security groups: operation error EC2: DescribeSecurityGroups, context deadline exceeded