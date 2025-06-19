Create EKS Cluster on AWS with ALB Ingress

## 🔹 Prerequisites

- ✅ IAM user with **Access Key ID** and **Secret Access Key**
- ✅ AWS CLI should be installed and configured

## 🔹 Install AWS CLI

```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
aws configure

```

## 🔹 Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

## 🔹 Install eksctl

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version

```

## ⚙️ Create EKS Cluster

```
eksctl create cluster --name=demo-cluster \
                    --region=us-east-2 \
                    --version=1.30 \
                    --name alb-sample-app \
                    --namespace game-2048 \
                    --without-nodegroup
```

## 🔹 Associate IAM OIDC Provider

```
eksctl utils associate-iam-oidc-provider \
  --region us-east-2 \
  --cluster demo-cluster \
  --approve
```

## 🔹 Create Node Group

```
eksctl create nodegroup --cluster=demo-cluster \
                     --region=us-east-2 \
                     --name=game-2048 \
                     --node-type=t2.large \
                     --nodes=2 \
                     --nodes-min=2 \
                     --nodes-max=2 \
                     --node-volume-size=29 \
                     --ssh-access \
                     --ssh-public-key=eks-nodegroup-key

```

> 🔑 Make sure the SSH key pair `eks-nodegroup-key` exists in your AWS account.

## 🚀 Deploy Application & Ingress

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

## 🛠️ Setup AWS Load Balancer Controller

### 🔹 Download IAM Policy

```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

### 🔹 Create IAM Policy

```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

### 🔹 Create IAM Service Account

```
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

### 🔹 Install AWS Load Balancer Controller via Helm

- Add Helm repo:

```
helm repo add eks https://aws.github.io/eks-charts
```

- Update repo:

```
helm repo update eks
```

- Install controller:

```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region> \
  --set vpcId=<your-vpc-id>
```

## ✅ Verification

- Check controller deployment:

```
kubectl get deployment -n kube-system aws-load-balancer-controller
```

- Verify ingress is created:

```
kubectl get ingress -n game-2048

```
