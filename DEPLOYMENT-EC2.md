# AWS EC2 + k3s Deployment Guide

**Self-Managed Lightweight Kubernetes on AWS EC2**

---

## 📌 Overview

This guide covers deploying the `auth-service` microservice on AWS EC2 using **k3s** (lightweight Kubernetes). This approach provides full infrastructure control at a lower cost compared to managed services.

### Use Cases
- **Development & Staging** environments
- **Cost-sensitive** production workloads
- **Learning** Kubernetes fundamentals
- **Small to Medium** scale applications

---

## Prerequisites

### AWS Account & EC2 Setup

1. **AWS Account** with EC2 access
2. **EC2 Instance Configuration**
   - **AMI:** Ubuntu 22.04 LTS
   - **Instance Type:** t3.medium or larger (2 vCPU, 4GB RAM minimum)
   - **Storage:** 50GB EBS volume (gp3 recommended)
   - **Network:** Public IP, VPC with internet gateway
   
3. **Security Group Rules**
   ```
   - 22/TCP    → SSH (your IP)
   - 80/TCP    → HTTP (0.0.0.0/0)
   - 443/TCP   → HTTPS (0.0.0.0/0)
   - 6443/TCP  → Kubernetes API (restricted)
   - 30000-32767/TCP → NodePort range (0.0.0.0/0)
   ```

4. **IAM Permissions**
   - EC2 instance requires IAM role with S3 access (if using for backups)

---

## 🔧 Installation Steps

### Step 1: Launch EC2 Instance

```bash
# Via AWS CLI
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.medium \
  --key-name your-key-pair \
  --security-group-ids sg-xxxxxxxx \
  --block-device-mappings DeviceName=/dev/sda1,Ebs={VolumeSize=50,VolumeType=gp3} \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=auth-service-k3s}]'
```

### Step 2: Connect to Instance

```bash
ssh -i your-key.pem ubuntu@EC2_PUBLIC_IP
```

### Step 3: Update System

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git jq net-tools htop
```

### Step 4: Install Docker

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group
sudo usermod -aG docker ubuntu
newgrp docker

# Verify installation
docker ps
```

### Step 5: Install k3s

```bash
# Install k3s with recommended settings
curl -sfL https://get.k3s.io | sh -s - \
  --write-kubeconfig-mode 644 \
  --disable servicelb \
  --disable traefik

# Verify installation
kubectl get nodes
kubectl get pods -A
```

### Step 6: Configure kubectl Access

```bash
# Setup kubeconfig for current user
mkdir -p $HOME/.kube
sudo cp /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
sudo chown $USER:$USER $HOME/.kube/config
chmod 600 $HOME/.kube/config

# Add to shell profile
echo 'export KUBECONFIG=$HOME/.kube/config' >> ~/.bashrc
source ~/.bashrc

# Test access
kubectl cluster-info
```

---

## 🐳 Build & Push Docker Image

### Step 1: Clone Repository

```bash
mkdir -p /var/www/ades/services
cd /var/www/ades/services
git clone https://github.com/YOUR_ORG/auth-service.git
cd auth-service
```

### Step 2: Build Docker Image

```bash
# Build image
docker build -t ynvenovore/ades-auth:1.0.1 .

# Test locally
docker run --rm -p 3026:3026 ynvenovore/ades-auth:1.0.1
```

### Step 3: Push to Docker Hub

```bash
# Authenticate with Docker Hub
docker login

# Push image
docker push ynvenovore/ades-auth:1.0.1
```

---

## ☸️ Kubernetes Deployment

### Step 1: Create Namespace

**File: `apis-namespace.yaml`**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: apis
  labels:
    environment: production
    managed-by: manual
```

Apply:
```bash
kubectl apply -f apis-namespace.yaml
```

### Step 2: Create ConfigMap

**File: `auth-service-configmap.yaml`**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: auth-service-config
  namespace: apis
data:
  NODE_ENV: production
  LOG_LEVEL: "debug"
  PORT: "3026"
  API_PREFIX: "/api"
```

Apply:
```bash
kubectl apply -f auth-service-configmap.yaml
```

### Step 3: Deploy Microservice

**File: `auth-service-deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: apis
  labels:
    app: auth-service
    version: "1.0.1"
    managed-by: manual
spec:
  replicas: 2
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
      restartPolicy: Always
      containers:
      - name: auth-service
        image: ynvenovore/ades-auth:1.0.1
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 3026
          protocol: TCP
        env:
        - name: NODE_ENV
          valueFrom:
            configMapKeyRef:
              name: auth-service-config
              key: NODE_ENV
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: auth-service-config
              key: LOG_LEVEL
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
  type: NodePort
  selector:
    app: auth-service
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 3026
    nodePort: 30081

---
apiVersion: v1
kind: Service
metadata:
  name: auth-service-lb
  namespace: apis
  labels:
    app: auth-service
spec:
  type: LoadBalancer
  selector:
    app: auth-service
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 3026
```

Apply:
```bash
kubectl apply -f auth-service-deployment.yaml
```

### Step 4: Verify Deployment

```bash
# Check deployment
kubectl get deployment -n apis
kubectl get pods -n apis
kubectl get svc -n apis

# View detailed status
kubectl describe deployment auth-service -n apis
kubectl describe pods -n apis

# Check logs
kubectl logs -n apis deployment/auth-service --tail=50 -f
```

---

## 🌐 Service Access

### Internal Access (from EC2)

```bash
# Via NodePort
curl http://localhost:30081/api/docs

# Via ClusterIP (from within cluster)
curl http://auth-service-svc.apis.svc.cluster.local/api/docs
```

### External Access

```
http://EC2_PUBLIC_IP:30081/api/docs
```

### Via Load Balancer

```bash
# Get LoadBalancer IP
kubectl get svc -n apis auth-service-lb

# Access via LoadBalancer
curl http://LOAD_BALANCER_IP/api/docs
```

---

## 📊 Monitoring Setup

### Install Prometheus

```bash
# Add Prometheus Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set prometheus.service.type=NodePort \
  --set prometheus.service.nodePort=30090
```

### Install Grafana

```bash
# Get default password
kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode

# Access Grafana
# http://EC2_PUBLIC_IP:30091 (default: admin/password)
```

---

## 🔧 Troubleshooting

### Issue: Pods in CrashLoopBackOff

```bash
# Check pod logs
kubectl logs -n apis deployment/auth-service

# Describe pod for events
kubectl describe pod -n apis <pod-name>

# Check resource usage
kubectl top nodes
kubectl top pods -n apis
```

### Issue: Service Not Accessible

```bash
# Verify service is running
kubectl get svc -n apis

# Check endpoints
kubectl get endpoints -n apis

# Test pod connectivity
kubectl run -it --rm test-pod --image=busybox -- sh
# Inside pod: wget http://auth-service-svc.apis.svc.cluster.local
```

### Issue: ImagePullBackOff

```bash
# Check image availability
docker images | grep ades-auth

# If needed, retag and push
docker tag auth-service:latest ynvenovore/ades-auth:1.0.1
docker push ynvenovore/ades-auth:1.0.1

# Restart deployment
kubectl rollout restart deployment/auth-service -n apis
```

---

## 🔐 Security Hardening

### Enable Network Policies

```yaml
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
    - namespaceSelector:
        matchLabels:
          name: apis
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

### Apply RBAC

```bash
# Create service account
kubectl create serviceaccount auth-service -n apis

# Create role
kubectl create role auth-service-role -n apis --verb=get,list,watch --resource=pods

# Bind role to service account
kubectl create rolebinding auth-service-binding -n apis \
  --role=auth-service-role --serviceaccount=apis:auth-service
```

---

## 🚀 Scaling & Updates

### Manual Scaling

```bash
# Scale deployment
kubectl scale deployment auth-service -n apis --replicas=3

# Watch scaling
kubectl get pods -n apis -w
```

### Rolling Update

```bash
# Update image
kubectl set image deployment/auth-service \
  auth-service=ynvenovore/ades-auth:1.0.2 -n apis

# Monitor rollout
kubectl rollout status deployment/auth-service -n apis

# Rollback if needed
kubectl rollout undo deployment/auth-service -n apis
```

### Auto-scaling (HPA)

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
  minReplicas: 2
  maxReplicas: 5
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

---

## 📋 Backup & Restore

### Backup Cluster

```bash
# Backup kubeconfig
cp ~/.kube/config ~/kubeconfig.backup

# Backup etcd (k3s uses SQLite)
sudo cp /var/lib/rancher/k3s/server/db/state.db ~/k3s-backup.db

# Backup manifests
kubectl get all -A -o yaml > ~/k3s-all-resources.yaml
```

### Restore Cluster

```bash
# Restore from backup
sudo cp ~/k3s-backup.db /var/lib/rancher/k3s/server/db/state.db
sudo systemctl restart k3s
```

---

## 🧹 Cleanup

```bash
# Delete deployment
kubectl delete deployment auth-service -n apis

# Delete service
kubectl delete svc auth-service-svc -n apis

# Delete namespace
kubectl delete namespace apis

# Uninstall k3s from EC2
/usr/local/bin/k3s-uninstall.sh
```

---

## 📚 References

- [k3s Documentation](https://docs.k3s.io)
- [Kubernetes Documentation](https://kubernetes.io/docs)
- [AWS EC2 Documentation](https://docs.aws.amazon.com/ec2)
- [Docker Documentation](https://docs.docker.com)

---

## 📋 Checklist

- [ ] EC2 instance created and security group configured
- [ ] Docker installed and running
- [ ] k3s installed and kubectl configured
- [ ] Docker image built and pushed
- [ ] Kubernetes namespace created
- [ ] Deployment applied and verified
- [ ] Service accessible internally and externally
- [ ] Monitoring configured
- [ ] Backup strategy implemented
- [ ] Scaling configured

---

**Status:** ✅ Production Ready | Last Updated: 2025-11-13
