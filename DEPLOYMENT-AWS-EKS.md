# AWS EKS Deployment Guide

**Enterprise-Grade Kubernetes on AWS**

---

## 📌 Overview

This guide covers deploying the `auth-service` microservice on **AWS EKS** (Elastic Kubernetes Service) - a fully managed Kubernetes service. EKS handles control plane management, auto-scaling, and integrates with AWS services.

### Key Benefits
- ✅ AWS-managed Kubernetes control plane
- ✅ Multi-AZ high availability
- ✅ Integrated with AWS IAM, VPC, CloudWatch
- ✅ Auto Scaling Groups for node management
- ✅ Application Load Balancer (ALB) integration
- ✅ VPC CNI for advanced networking

---

## Prerequisites

### AWS Account Setup

1. **AWS Account** with appropriate permissions
2. **IAM User/Role** with policies:
   - `AmazonEKSFullAccess`
   - `AmazonEC2FullAccess`
   - `AmazonVPCFullAccess`
   - `CloudWatchLogsFullAccess`

3. **AWS CLI** installed and configured
   ```bash
   aws --version
   aws configure
   ```

4. **kubectl** installed
   ```bash
   curl -LO https://dl.k8s.io/release/stable.txt
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   chmod +x kubectl && sudo mv kubectl /usr/local/bin/
   ```

5. **eksctl** - AWS EKS cluster management
   ```bash
   curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
   sudo mv /tmp/eksctl /usr/local/bin
   eksctl version
   ```

6. **Helm** - Package manager
   ```bash
   curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
   ```

---

## 🏗️ AWS Infrastructure Setup

### Step 1: Create VPC (Optional - eksctl creates default)

For production, create a dedicated VPC:

```bash
# Create VPC with public and private subnets
aws ec2 create-vpc --cidr-block 10.0.0.0/16

# Note the VpcId, then create subnets
# Public subnets for NAT gateways and load balancers
# Private subnets for EKS nodes
```

### Step 2: Create EKS Cluster

#### Option A: Using eksctl (Recommended)

```bash
# Create cluster with all necessary resources
eksctl create cluster \
  --name ades-auth-cluster \
  --version 1.28 \
  --region us-east-1 \
  --nodegroup-name ades-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 5 \
  --managed \
  --enable-ssm \
  --with-oidc

# This takes ~15 minutes
```

#### Option B: Using AWS Console

```bash
# Via AWS Management Console:
# 1. Navigate to EKS → Clusters → Create cluster
# 2. Select "Create cluster"
# 3. Cluster name: ades-auth-cluster
# 4. Version: 1.28 (or latest)
# 5. Role ARN: Create new service role
# 6. VPC: Select or create VPC
# 7. Subnets: Select public subnets
# 8. Security groups: Create/select security group
# 9. Logging: Enable control plane logs
# 10. Create node group with desired settings
```

### Step 3: Configure kubectl

```bash
# Update kubeconfig
aws eks update-kubeconfig \
  --region us-east-1 \
  --name ades-auth-cluster

# Verify access
kubectl cluster-info
kubectl get nodes
```

---

## 🔐 IAM & RBAC Setup

### Create IAM Role for Service Account

```bash
# Enable OIDC provider
eksctl utils associate-iam-oidc-provider \
  --cluster=ades-auth-cluster \
  --region=us-east-1 \
  --approve

# Create IAM role for auth-service
eksctl create iamserviceaccount \
  --name auth-service-sa \
  --namespace apis \
  --cluster ades-auth-cluster \
  --region us-east-1 \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess \
  --approve
```

### Apply RBAC

```yaml
# rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: auth-service
  namespace: apis
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/auth-service-role

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: auth-service-role
  namespace: apis
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: auth-service-rolebinding
  namespace: apis
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: auth-service-role
subjects:
- kind: ServiceAccount
  name: auth-service
  namespace: apis
```

Apply:
```bash
kubectl apply -f rbac.yaml
```

---

## 🐳 Build & Push Docker Image

### Push to Amazon ECR

```bash
# Create ECR repository
aws ecr create-repository --repository-name ades-auth --region us-east-1

# Get login token
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com

# Build and tag image
docker build -t ades-auth:1.0.1 .
docker tag ades-auth:1.0.1 \
  ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/ades-auth:1.0.1

# Push to ECR
docker push ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/ades-auth:1.0.1
```

---

## ☸️ Kubernetes Deployment

### Step 1: Create Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: apis
  labels:
    environment: production
```

### Step 2: Create Secret for Database (Example)

```bash
# Create secret for sensitive data
kubectl create secret generic auth-service-secrets \
  --from-literal=DB_PASSWORD=your_password \
  --from-literal=JWT_SECRET=your_jwt_secret \
  -n apis
```

### Step 3: Deploy with Deployment & Service

**File: `auth-service-eks.yaml`**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: apis

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: auth-service-config
  namespace: apis
data:
  NODE_ENV: "production"
  LOG_LEVEL: "info"
  PORT: "3026"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: apis
  labels:
    app: auth-service
    version: "1.0.1"
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3026"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: auth-service
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
      - name: auth-service
        image: ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/ades-auth:1.0.1
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 3026
          protocol: TCP
        envFrom:
        - configMapRef:
            name: auth-service-config
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: auth-service-secrets
              key: DB_PASSWORD
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3026
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: 3026
          initialDelaySeconds: 10
          periodSeconds: 5
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
              - ALL

---
apiVersion: v1
kind: Service
metadata:
  name: auth-service-svc
  namespace: apis
  labels:
    app: auth-service
spec:
  type: ClusterIP
  selector:
    app: auth-service
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 3026

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: auth-service-netpol
  namespace: apis
spec:
  podSelector:
    matchLabels:
      app: auth-service
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 3026
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 443
```

Apply:
```bash
kubectl apply -f auth-service-eks.yaml
```

### Step 3: Setup Ingress with ALB

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auth-service-ingress
  namespace: apis
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:ACCOUNT_ID:certificate/XXXXX
spec:
  ingressClassName: alb
  rules:
  - host: auth-api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: auth-service-svc
            port:
              number: 80
```

First, install AWS Load Balancer Controller:

```bash
# Add AWS Load Balancer Controller Helm repo
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Install ALB controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=ades-auth-cluster \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller
```

---

## 📊 Monitoring & Logging

### Enable CloudWatch Container Insights

```bash
# Create IAM policy for CloudWatch
curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-yaml/cwagent/fluentd/iam-policy.json -o cwagent-policy.json

# Create IAM role
aws iam create-role --role-name CloudWatchAgentRole \
  --assume-role-policy-document file://trust-policy.json

# Attach policy
aws iam put-role-policy --role-name CloudWatchAgentRole \
  --policy-name CloudWatchAgentPolicy \
  --policy-document file://cwagent-policy.json

# Deploy Container Insights
curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-yaml/container-insights-monitoring.yaml | \
  sed "s/{{cluster_name}}/ades-auth-cluster/;s/{{region_name}}/us-east-1/" | kubectl apply -f -
```

### Install Prometheus & Grafana

```bash
# Add Prometheus Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set prometheus.service.type=LoadBalancer
```

---

## 🚀 Auto-Scaling

### Pod Auto-Scaling (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: auth-service-hpa
  namespace: apis
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: auth-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### Node Auto-Scaling (Cluster Autoscaler)

```bash
# Install Cluster Autoscaler
helm install autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=ades-auth-cluster \
  --set awsRegion=us-east-1
```

---

## 🧹 Cleanup

```bash
# Delete resources
kubectl delete namespace apis
kubectl delete ingress auth-service-ingress -n apis

# Delete cluster (takes ~15 minutes)
eksctl delete cluster --name ades-auth-cluster --region us-east-1
```

---

## 📚 References

- [AWS EKS Documentation](https://docs.aws.amazon.com/eks)
- [eksctl Documentation](https://eksctl.io)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller)
- [Container Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights.html)

---

**Status:** ✅ Enterprise Ready | Last Updated: 2025-11-13
