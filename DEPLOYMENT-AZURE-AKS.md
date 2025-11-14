# Microsoft Azure AKS Deployment Guide

**Enterprise Kubernetes on Microsoft Azure**

---

## 📌 Overview

This guide covers deploying the `auth-service` microservice on **Azure AKS** (Azure Kubernetes Service) - a fully managed Kubernetes service integrated with Microsoft Azure ecosystem. AKS provides enterprise-grade security, compliance, and integration with Azure services.

### Key Benefits
- ✅ Microsoft-managed Kubernetes control plane
- ✅ Multi-zone high availability
- ✅ Managed Identities for secure authentication
- ✅ Azure AD integration
- ✅ Azure Load Balancing
- ✅ Azure Monitor & Log Analytics
- ✅ Container Registry (ACR) integration
- ✅ Network Security Groups

---

## Prerequisites

### Azure Subscription Setup

1. **Azure Subscription**
   ```bash
   az account list
   az account set --subscription "Subscription Name"
   ```

2. **Azure CLI** installed
   ```bash
   curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
   az version
   ```

3. **kubectl** installed
   ```bash
   az aks install-cli
   ```

4. **Helm** installed
   ```bash
   curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
   ```

5. **Required Permissions**
   - `Subscription Contributor` or custom role with AKS permissions

---

## 🏗️ Azure Infrastructure Setup

### Step 1: Create Resource Group

```bash
# Create resource group
az group create \
  --name ades-auth-rg \
  --location eastus

export RESOURCE_GROUP=ades-auth-rg
export LOCATION=eastus
```

### Step 2: Create Virtual Network (Optional)

```bash
# Create VNet
az network vnet create \
  --resource-group $RESOURCE_GROUP \
  --name ades-auth-vnet \
  --address-prefix 10.0.0.0/16 \
  --subnet-name ades-auth-subnet \
  --subnet-prefix 10.0.0.0/24

# Get subnet ID
SUBNET_ID=$(az network vnet subnet show \
  --resource-group $RESOURCE_GROUP \
  --vnet-name ades-auth-vnet \
  --name ades-auth-subnet \
  --query id -o tsv)
```

### Step 3: Create Container Registry

```bash
# Create Azure Container Registry (ACR)
az acr create \
  --resource-group $RESOURCE_GROUP \
  --name adesauthregistry \
  --sku Standard

# Get registry URL
REGISTRY_URL=$(az acr show \
  --resource-group $RESOURCE_GROUP \
  --name adesauthregistry \
  --query loginServer -o tsv)
```

### Step 4: Create AKS Cluster

```bash
# Create AKS cluster
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name ades-auth-cluster \
  --node-count 2 \
  --vm-set-type VirtualMachineScaleSets \
  --load-balancer-sku standard \
  --enable-managed-identity \
  --network-plugin azure \
  --vnet-subnet-id $SUBNET_ID \
  --docker-bridge-address 172.17.0.1/16 \
  --service-cidr 10.1.0.0/16 \
  --dns-service-ip 10.1.0.10 \
  --enable-cluster-autoscaling \
  --min-count 2 \
  --max-count 5 \
  --enable-aad \
  --aad-server-app-id <AAD_SERVER_APP_ID> \
  --aad-server-app-secret <AAD_SERVER_APP_SECRET> \
  --aad-client-app-id <AAD_CLIENT_APP_ID> \
  --enable-azure-policy \
  --enable-log-analytics-workspace \
  --workspace-resource-id <LOG_ANALYTICS_WORKSPACE_ID> \
  --zones 1 2 3
```

### Step 5: Configure kubectl

```bash
# Get cluster credentials
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name ades-auth-cluster \
  --admin

# Verify access
kubectl cluster-info
kubectl get nodes
```

---

## 🔐 Managed Identities Setup

### Create User-Assigned Managed Identity

```bash
# Create managed identity
az identity create \
  --resource-group $RESOURCE_GROUP \
  --name auth-service-identity

# Get identity details
IDENTITY_ID=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name auth-service-identity \
  --query id -o tsv)

PRINCIPAL_ID=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name auth-service-identity \
  --query principalId -o tsv)
```

### Assign Roles

```bash
# Grant ACR pull role
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role AcrPull \
  --scope /subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.ContainerRegistry/registries/adesauthregistry

# Grant Azure Monitor role
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Monitoring Metrics Publisher"

# Grant Log Analytics role
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Log Analytics Contributor"
```

---

## 🐳 Build & Push Docker Image

### Push to Azure Container Registry

```bash
# Login to ACR
az acr login --name adesauthregistry

# Build image
docker build -t ades-auth:1.0.1 .

# Tag image
docker tag ades-auth:1.0.1 $REGISTRY_URL/ades-auth:1.0.1

# Push to ACR
docker push $REGISTRY_URL/ades-auth:1.0.1
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
# Create secret for sensitive data
kubectl create secret generic auth-service-secrets \
  --from-literal=DB_PASSWORD=your_password \
  --from-literal=JWT_SECRET=your_jwt_secret \
  -n apis
```

### Step 3: Deploy Application

**File: `auth-service-aks.yaml`**

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
  AZURE_SUBSCRIPTION_ID: "SUBSCRIPTION_ID"

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: auth-service
  namespace: apis
  annotations:
    azure.workload.identity/client-id: "CLIENT_ID"

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
        azure.workload.identity/use: "true"
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
        image: REGISTRY_URL/ades-auth:1.0.1
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
kubectl apply -f auth-service-aks.yaml
```

### Step 4: Setup Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auth-service-ingress
  namespace: apis
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - auth-api.example.com
    secretName: auth-service-tls
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

Install Application Gateway Ingress Controller:
```bash
# Add Helm repo
helm repo add application-gateway-kubernetes-ingress \
  https://appgwkic.blob.core.windows.net/helm/helm-repo/
helm repo update

# Install AGIC
helm install agic application-gateway-kubernetes-ingress/ingress-azure \
  --namespace kube-system \
  --set appgw.subscriptionId=$(az account show --query id -o tsv) \
  --set appgw.resourceGroup=$RESOURCE_GROUP \
  --set appgw.name=ades-auth-appgw \
  --set armAuth.type=aadPodIdentity \
  --set rbac.enabled=true
```

---

## 📊 Monitoring & Logging

### Azure Monitor Integration

```bash
# Enable Container Insights
az aks enable-addons \
  --resource-group $RESOURCE_GROUP \
  --name ades-auth-cluster \
  --addons monitoring

# Get workspace resource ID
WORKSPACE_ID=$(az aks show \
  --resource-group $RESOURCE_GROUP \
  --name ades-auth-cluster \
  --query addonProfiles.omsagent.config.logAnalyticsWorkspaceResourceID -o tsv)
```

### View Logs

```bash
# View live logs
kubectl logs -n apis deployment/auth-service --tail=50 -f

# View via Azure CLI
az monitor log-analytics query \
  --workspace $(echo $WORKSPACE_ID | xargs basename) \
  --analytics-query "ContainerLog | where Namespace == 'apis' | tail 50"
```

### Setup Alerts

```bash
# Create metric alert
az monitor metrics alert create \
  --name auth-service-cpu-high \
  --resource-group $RESOURCE_GROUP \
  --scopes $WORKSPACE_ID \
  --condition "avg Percentage CPU > 80" \
  --description "Alert when CPU usage exceeds 80%"
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

### Node Auto-Scaling

Automatic with Azure Virtual Machine Scale Sets. To customize:

```bash
# Update node pool
az aks nodepool update \
  --resource-group $RESOURCE_GROUP \
  --cluster-name ades-auth-cluster \
  --name nodepool1 \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 5
```

---

## 🔍 Troubleshooting

### Check Cluster Status

```bash
# Get cluster info
az aks show \
  --resource-group $RESOURCE_GROUP \
  --name ades-auth-cluster

# Check node status
az aks nodepool list \
  --resource-group $RESOURCE_GROUP \
  --cluster-name ades-auth-cluster
```

### Check Pod Status

```bash
# Get pods
kubectl get pods -n apis

# Describe pod
kubectl describe pod -n apis <pod-name>

# View logs
kubectl logs -n apis deployment/auth-service --tail=50 -f
```

### Check Managed Identity

```bash
# Verify identity is assigned
kubectl get serviceaccount auth-service -n apis -o yaml

# Test identity access
kubectl run -it --rm test-pod \
  --image=mcr.microsoft.com/azure-cli:latest \
  --serviceaccount=auth-service \
  -n apis \
  -- az account show
```

---

## 🧹 Cleanup

```bash
# Delete resources
kubectl delete namespace apis

# Delete resource group (deletes everything)
az group delete \
  --resource-group $RESOURCE_GROUP \
  --yes --no-wait
```

---

## 📚 References

- [Azure AKS Documentation](https://docs.microsoft.com/en-us/azure/aks)
- [Workload Identity Documentation](https://azure.microsoft.com/en-us/blog/azure-workload-identity-faq)
- [Azure Container Registry](https://docs.microsoft.com/en-us/azure/container-registry)
- [Azure Monitor for Containers](https://docs.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-overview)

---

**Status:** ✅ Enterprise Ready | Last Updated: 2025-11-13
