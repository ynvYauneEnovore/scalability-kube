# Google Cloud GKE Deployment Guide

**Enterprise Kubernetes on Google Cloud Platform**

---

## 📌 Overview

This guide covers deploying the `auth-service` microservice on **Google Cloud GKE** (Google Kubernetes Engine) - a fully managed Kubernetes service optimized for containerized applications. GKE offers seamless integration with Google Cloud services and advanced features like Workload Identity.

### Key Benefits
- ✅ Google-managed Kubernetes control plane
- ✅ Multi-zone high availability
- ✅ Workload Identity for secure authentication
- ✅ GCP IAM integration
- ✅ Cloud Load Balancing
- ✅ Cloud Logging & Cloud Trace integration
- ✅ Autopilot clusters (fully managed nodes)

---

## Prerequisites

### Google Cloud Project Setup

1. **Google Cloud Project**
   ```bash
   gcloud projects create ades-auth-project --name="ADES Auth Service"
   export PROJECT_ID=$(gcloud config get-value project)
   ```

2. **Enable Required APIs**
   ```bash
   gcloud services enable container.googleapis.com
   gcloud services enable compute.googleapis.com
   gcloud services enable artifactregistry.googleapis.com
   gcloud services enable cloudlogging.googleapis.com
   gcloud services enable monitoring.googleapis.com
   ```

3. **gcloud CLI** installed
   ```bash
   curl https://sdk.cloud.google.com | bash
   gcloud init
   ```

4. **kubectl** installed
   ```bash
   gcloud components install kubectl
   ```

5. **Helm** installed
   ```bash
   curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
   ```

---

## 🏗️ GCP Infrastructure Setup

### Step 1: Create GKE Cluster

#### Option A: Using gcloud CLI (Standard Cluster)

```bash
# Create cluster with best practices
gcloud container clusters create ades-auth-cluster \
  --region us-central1 \
  --num-nodes 2 \
  --machine-type n1-standard-2 \
  --disk-size 50 \
  --enable-autoscaling \
  --min-nodes 2 \
  --max-nodes 5 \
  --enable-ip-alias \
  --network "default" \
  --enable-stackdriver-kubernetes \
  --enable-workload-identity \
  --enable-network-policy \
  --addons HttpLoadBalancing,HttpsLoadBalancing,HorizontalPodAutoscaling \
  --workload-pool=${PROJECT_ID}.svc.id.goog
```

#### Option B: Using Autopilot Cluster (Google-Managed Nodes)

```bash
# Autopilot handles node management completely
gcloud container clusters create-auto ades-auth-cluster \
  --region us-central1 \
  --enable-workload-identity \
  --enable-network-policy
```

### Step 2: Configure kubectl

```bash
# Get cluster credentials
gcloud container clusters get-credentials ades-auth-cluster \
  --region us-central1

# Verify access
kubectl cluster-info
kubectl get nodes
```

---

## 🔐 Workload Identity Setup

### Create Service Account

```bash
# Create Kubernetes service account
kubectl create namespace apis
kubectl create serviceaccount auth-service -n apis

# Create Google Service Account
gcloud iam service-accounts create auth-service \
  --display-name="ADES Auth Service"

# Get service account email
export GSA_EMAIL=$(gcloud iam service-accounts list --filter="displayName:ADES Auth Service" --format='value(email)')
```

### Bind Service Accounts

```bash
# Grant Workload Identity binding
gcloud iam service-accounts add-iam-policy-binding $GSA_EMAIL \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:${PROJECT_ID}.svc.id.goog[apis/auth-service]"

# Annotate Kubernetes service account
kubectl annotate serviceaccount auth-service -n apis \
  iam.gke.io/gcp-service-account=$GSA_EMAIL
```

### Grant IAM Permissions

```bash
# Grant necessary permissions
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$GSA_EMAIL \
  --role=roles/logging.logWriter

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$GSA_EMAIL \
  --role=roles/monitoring.metricWriter

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$GSA_EMAIL \
  --role=roles/cloudtrace.agent
```

---

## 🐳 Build & Push Docker Image

### Push to Google Artifact Registry

```bash
# Create Artifact Registry repository
gcloud artifacts repositories create ades-auth-repo \
  --repository-format=docker \
  --location=us-central1 \
  --description="ADES Auth Service Docker Images"

# Configure authentication
gcloud auth configure-docker us-central1-docker.pkg.dev

# Build image
docker build -t ades-auth:1.0.1 .

# Tag image
docker tag ades-auth:1.0.1 \
  us-central1-docker.pkg.dev/${PROJECT_ID}/ades-auth-repo/ades-auth:1.0.1

# Push to Artifact Registry
docker push us-central1-docker.pkg.dev/${PROJECT_ID}/ades-auth-repo/ades-auth:1.0.1
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

### Step 2: Create Secrets

```bash
# Create secret for sensitive configuration
kubectl create secret generic auth-service-secrets \
  --from-literal=DB_PASSWORD=your_password \
  --from-literal=JWT_SECRET=your_jwt_secret \
  -n apis
```

### Step 3: Deploy Application

**File: `auth-service-gke.yaml`**

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
  GOOGLE_CLOUD_PROJECT: "PROJECT_ID"

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
        image: us-central1-docker.pkg.dev/PROJECT_ID/ades-auth-repo/ades-auth:1.0.1
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
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: auth-service-secrets
              key: JWT_SECRET
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
kubectl apply -f auth-service-gke.yaml
```

### Step 4: Setup Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auth-service-ingress
  namespace: apis
  annotations:
    kubernetes.io/ingress.global-static-ip-name: auth-service-ip
    networking.gke.io/managed-certificates: auth-service-cert
    kubernetes.io/ingress.class: "gce"
spec:
  rules:
  - host: auth-api.example.com
    http:
      paths:
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: auth-service-svc
            port:
              number: 80

---
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: auth-service-cert
  namespace: apis
spec:
  domains:
  - auth-api.example.com
```

Reserve static IP:
```bash
gcloud compute addresses create auth-service-ip --global
gcloud compute addresses describe auth-service-ip --global
```

Apply ingress:
```bash
kubectl apply -f ingress-gke.yaml
```

---

## 📊 Monitoring & Logging

### Cloud Logging Integration

```bash
# Cloud Logging is automatically enabled
# View logs via gcloud
gcloud logging read "resource.type=k8s_container AND namespace_name=apis"

# Or view in Cloud Console
# https://console.cloud.google.com/logs
```

### Install Prometheus

```bash
# Add Prometheus Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

### Setup Cloud Monitoring

```bash
# Create dashboard
gcloud monitoring dashboards create --config-from-file=- <<EOF
{
  "displayName": "ADES Auth Service",
  "gridLayout": {
    "widgets": [
      {
        "title": "Pod CPU Usage",
        "xyChart": {
          "dataSets": [
            {
              "timeSeriesQuery": {
                "timeSeriesFilter": {
                  "filter": "metric.type=\"kubernetes.io/pod/cpu/core_usage_time\" resource.namespace_name=\"apis\""
                }
              }
            }
          ]
        }
      }
    ]
  }
}
EOF
```

---

## 🚀 Auto-Scaling

### Pod Auto-Scaling

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

### Node Auto-Scaling (Automatic in GKE)

GKE automatically scales nodes. To customize:

```bash
# Update node pool autoscaling
gcloud container node-pools update default-pool \
  --cluster=ades-auth-cluster \
  --region=us-central1 \
  --enable-autoscaling \
  --min-nodes=2 \
  --max-nodes=5
```

---

## 🔍 Troubleshooting

### Check Pod Status

```bash
# Get pods
kubectl get pods -n apis

# Describe pod
kubectl describe pod -n apis <pod-name>

# View logs
kubectl logs -n apis deployment/auth-service --tail=50 -f
```

### Check Workload Identity

```bash
# Verify service account annotation
kubectl get serviceaccount -n apis auth-service -o yaml

# Test Workload Identity
kubectl run -it --rm test-pod \
  --image=google/cloud-sdk:slim \
  --serviceaccount=auth-service \
  -n apis \
  -- gcloud auth list
```

---

## 🧹 Cleanup

```bash
# Delete resources
kubectl delete namespace apis

# Delete cluster
gcloud container clusters delete ades-auth-cluster --region us-central1

# Delete Artifact Registry repository
gcloud artifacts repositories delete ades-auth-repo --location=us-central1

# Delete static IP
gcloud compute addresses delete auth-service-ip --global
```

---

## 📚 References

- [Google Cloud GKE Documentation](https://cloud.google.com/kubernetes-engine/docs)
- [Workload Identity Documentation](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)
- [GKE Best Practices](https://cloud.google.com/kubernetes-engine/docs/best-practices)
- [Cloud Logging Documentation](https://cloud.google.com/logging/docs)

---

**Status:** ✅ Enterprise Ready | Last Updated: 2025-11-13
