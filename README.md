# Kubernetes Infrastructure

## Prerequisites and Preparation infrastructure

* Docs
    * awscli https://docs.aws.amazon.com/cli/latest/userguide/aws-cli.pdf
    * eksctl https://eksctl.io/
    * ebs https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/deploy/installation/
    * efs https://github.com/kubernetes-sigs/aws-efs-csi-driver
    * nginx ingress https://kubernetes.github.io/ingress-nginx/deploy/


## Prerequisites

```bash
brew install awscli
brew install weaveworks/tap/eksctl
brew upgrade eksctl && brew link --overwrite eksctl

# Configuration aws key
aws configuration

# Create cluster
eksctl create cluster -f k8s/cluster.yaml

# Install Helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

## Preparation

```bash
# Add EKS Charts
helm repo add eks https://aws.github.io/eks-charts

# Add Load Balancer Helm Repesitory
helm install aws-load-balancer-controller eks/aws-load-balancer-controller --set clusterName=nameCluster -n kube-system

# Install nginx ingress controller
# Edit the file and change
# VPC CIDR in use for the Kubernetes cluster:
# proxy-real-ip-cidr: XXX.XXX.XXX/XX or 0.0.0.0/0
# AWS Certificate Manager (ACM) ID
# arn:aws:acm:us-west-2:XXXXXXXX:certificate/XXXXXX-XXXXXXX-XXXXXXX-XXXXXXXX
kubectl apply -f k8s/deploy-tls-termination.yaml
kubectl get svc -n ingress-nginx

# Custom config nginx
kubectl apply -f k8s/custom-config-nginx.yaml

# Install EFS 
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm repo update
helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
    --namespace kube-system \
    --set image.repository=602401143452.dkr.ecr.ap-southeast-1.amazonaws.com/eks/aws-efs-csi-driver \
    --set controller.serviceAccount.create=false \
    --set controller.serviceAccount.name=efs-csi-controller-sa

# Create image pull secret
kubectl create secret docker-registry ecr-secret \
  --docker-server=xxxxxxx.dkr.ecr.ap-southeast-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password) \
  --namespace=dev

kubectl get secret ecr-secret --namespace=dev -oyaml | kubectl apply --namespace=uat -f -
kubectl get secret ecr-secret --namespace=dev -oyaml | kubectl apply --namespace=prd -f -
```