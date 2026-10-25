# Deployment Steps

## 1. Install the Tools

kubectl (PowerShell):

```powershell
winget install -e --id Kubernetes.kubectl
```

eksctl (Git Bash, official AWS method):

```bash
ARCH=amd64
PLATFORM=windows_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.zip"
unzip eksctl_$PLATFORM.zip -d "$HOME/bin"
rm eksctl_$PLATFORM.zip
```

AWS CLI (PowerShell, official install script):

```powershell
irm https://awscli.amazonaws.com/v2/install.ps1 | iex
```

Verify:

```bash
kubectl version --client
eksctl version
aws --version
```

## 2. Configure the AWS CLI

```bash
aws configure
aws sts get-caller-identity
```

## 3. Create the EKS Cluster

```bash
eksctl create cluster --name demo-cluster --region eu-central-1 --fargate
```

```bash
aws eks update-kubeconfig --name demo-cluster --region eu-central-1
kubectl config current-context
kubectl get nodes
```

## 4. Deploy the 2048 App

Create Fargate Profile:

```bash
eksctl create fargateprofile \
    --cluster demo-cluster \
    --region eu-central-1 \
    --name alb-sample-app \
    --namespace game-2048
```

Deploy the deployment, service and Ingress:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

Verify: 

```bash
kubectl get pods -n game-2048
kubectl get svc -n game-2048
kubectl get ingress -n game-2048
```

## 5. Configure the IAM OIDC Provider

```bash
export cluster_name=demo-cluster
oidc_id=$(aws eks describe-cluster --name $cluster_name --region eu-central-1 --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
echo $oidc_id
```

Check if an IAM OIDC provider is already configured:

```bash
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```

No output means no provider exists yet, so create one:

```bash
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --region eu-central-1 --approve
```

## 6. Install the AWS Load Balancer Controller

Download the IAM policy the controller needs:

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

Create the IAM policy in AWS:

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

Create the IAM role and the Kubernetes service account for the controller:

```bash
eksctl create iamserviceaccount \
  --cluster=demo-cluster \
  --region=eu-central-1 \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

Add and update the EKS Helm chart repository:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```

Get the cluster's VPC ID (needed for the install):

```bash
aws eks describe-cluster --name demo-cluster --region eu-central-1 --query "cluster.resourcesVpcConfig.vpcId" --output text
```

Install the controller with Helm:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=eu-central-1 \
  --set vpcId=<your-vpc-id>
```

Verify the controller is running:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

## 7. Access the App

```bash
kubectl get ingress -n game-2048
```

Open the address from the `ADDRESS` column with `http://`.

## 8. Add HTTPS with a Custom Domain

Request a public certificate in **AWS Certificate Manager** (same region as the ALB, `eu-central-1`) for your subdomain, e.g. `2048.poppy-gmbh.site`, using **DNS validation**.

Add the CNAME record shown in ACM to your DNS provider (GoDaddy). For the name, leave out the domain part at the end, since GoDaddy adds it automatically. Check that the record is live:

```bash
nslookup -type=CNAME <cname-name-from-acm>
```

Once the certificate status is **Issued**, apply the HTTPS Ingress (with your certificate ARN in `ingress-https.yaml`):

```bash
kubectl apply -f ingress-https.yaml
```

Verify that the controller applied the changes (look for `Successfully reconciled`):

```bash
kubectl describe ingress ingress-2048 -n game-2048
```

Point the subdomain to the ALB with a CNAME record in your DNS provider:

- **Name:** `2048`
- **Value:** the address from `kubectl get ingress -n game-2048`

Check the HTTP to HTTPS redirect:

```bash
curl -I http://2048.poppy-gmbh.site
```

The app is now available at `https://2048.poppy-gmbh.site`.

## 9. Clean Up

Delete the app first, so the AWS Load Balancer Controller removes the ALB:

```bash
kubectl delete -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

Wait a few minutes, then delete the cluster:

```bash
eksctl delete cluster --name demo-cluster --region eu-central-1
```

Then delete the ACM certificate and remove the two CNAME records from your DNS provider.