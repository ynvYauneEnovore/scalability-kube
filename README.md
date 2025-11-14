# Microservices Infrastructure

**Multi-Cloud Kubernetes Deployment Guide**

---

## 📋 Overview

This repository contains enterprise-grade deployment guides for the ADES microservices architecture across multiple cloud providers. The `auth-service` microservice (NestJS) can be deployed on:

- **AWS** - EC2 (k3s) or EKS (managed Kubernetes)
- **Google Cloud** - GKE (managed Kubernetes)
- **Microsoft Azure** - AKS (managed Kubernetes)

### Technology Stack

| Component | Purpose |
|-----------|---------|
| **Docker** | Container runtime and image management |
| **Kubernetes** | Container orchestration |
| **NestJS** | API framework and application runtime |
| **Helm** | Kubernetes package management |
| **Prometheus & Grafana** | Monitoring and observability |
| **Istio** | Service mesh (optional) |

---

## 🚀 Deployment Options

### Option 1: AWS EC2 + k3s (Self-Managed)
**Best for:** Cost-conscious teams, learning, development environments

📖 [See EC2 Deployment Guide](./DEPLOYMENT-EC2.md)

- Lightweight k3s cluster
- Full control over infrastructure
- Suitable for small to medium workloads

---

### Option 2: AWS EKS (Managed Kubernetes)
**Best for:** Enterprise, production-grade, auto-scaling requirements

📖 [See AWS EKS Deployment Guide](./DEPLOYMENT-AWS-EKS.md)

- AWS-managed control plane
- VPC integration, security groups, IAM roles
- Auto Scaling Groups, load balancing
- CloudWatch monitoring integration
- Multi-AZ high availability

---

### Option 3: Google Cloud GKE (Managed Kubernetes)
**Best for:** Organizations using Google Cloud ecosystem, ML workloads

📖 [See Google Cloud GKE Deployment Guide](./DEPLOYMENT-GCP-GKE.md)

- Google-managed control plane
- VPC networking, firewall rules
- Workload Identity for secure access
- Stackdriver monitoring integration
- Autopilot clusters (fully managed nodes)

---

### Option 4: Microsoft Azure AKS (Managed Kubernetes)
**Best for:** Enterprises with Azure/Microsoft infrastructure, hybrid cloud

📖 [See Azure AKS Deployment Guide](./DEPLOYMENT-AZURE-AKS.md)

- Microsoft-managed control plane
- Virtual Networks, Network Security Groups
- Managed Identities for secure access
- Azure Monitor integration
- Multi-tenant cluster management

---

## 🏗️ Architecture Comparison

| Feature | EC2 (k3s) | AWS EKS | GCP GKE | Azure AKS |
|---------|-----------|---------|---------|-----------|
| **Management** | Self-managed | AWS-managed | Google-managed | Azure-managed |
| **Control Plane** | Included | Managed (free) | Managed (free) | Managed (free) |
| **Cost** | Low | Medium | Medium | Medium |
| **Scalability** | Manual | Auto-scaling | Auto-scaling | Auto-scaling |
| **HA/DR** | Manual config | Built-in Multi-AZ | Built-in Multi-zone | Built-in Multi-AZ |
| **Monitoring** | Custom setup | CloudWatch | Stackdriver | Azure Monitor |
| **Security** | Manual hardening | IAM roles, security groups | Workload Identity | Managed Identities |
| **Learning Curve** | Medium | High | Medium | High |

---

## Quick Start

## Quick Start

### Prerequisites

Regardless of cloud provider, ensure you have:

1. **kubectl** - Kubernetes CLI
   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   chmod +x kubectl && sudo mv kubectl /usr/local/bin/
   ```

2. **Helm** - Package manager for Kubernetes
   ```bash
   curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
   ```

3. **Cloud CLI tools** (depending on your choice):
   - **AWS:** `aws-cli`
   - **Google Cloud:** `gcloud`
   - **Azure:** `az-cli`

4. **Docker** - Container runtime
   ```bash
   curl -fsSL https://get.docker.com -o get-docker.sh && sh get-docker.sh
   ```

---

## 🔐 Security Best Practices

Across all deployments:

- ✅ Use managed identities/service accounts
- ✅ Implement network policies and firewalls
- ✅ Enable RBAC (Role-Based Access Control)
- ✅ Use private registries for container images
- ✅ Implement pod security policies
- ✅ Enable encryption at rest and in transit
- ✅ Use secrets management (Vault, managed services)
- ✅ Regular security audits and compliance checks

---

## 📊 Monitoring & Observability

Each guide includes setup for:

- **Prometheus** - Metrics collection
- **Grafana** - Visualization dashboards
- **ELK Stack** or cloud-native logging
- **Distributed tracing** (Jaeger/Zipkin)

---

## 🛠️ Troubleshooting

Common issues and solutions are documented in each deployment guide specific to the platform.

For general Kubernetes troubleshooting:
```bash
kubectl describe pods -n apis
kubectl logs -n apis deployment/auth-service
kubectl get events -n apis --sort-by='.lastTimestamp'
```

---

## 📚 Documentation Structure

```
.
├── README.md                      # This file (overview)
├── DEPLOYMENT-EC2.md             # k3s on EC2 (self-managed)
├── DEPLOYMENT-AWS-EKS.md         # AWS EKS (managed)
├── DEPLOYMENT-GCP-GKE.md         # Google Cloud GKE (managed)
├── DEPLOYMENT-AZURE-AKS.md       # Azure AKS (managed)
├── manifests/
│   ├── base/
│   │   ├── namespace.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml
│   ├── aws-eks/
│   │   ├── ingress.yaml
│   │   └── storage.yaml
│   ├── gcp-gke/
│   │   ├── ingress.yaml
│   │   └── workload-identity.yaml
│   └── azure-aks/
│       ├── ingress.yaml
│       └── managed-identity.yaml
└── helm/
    └── ades-auth-service/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
```

---

## 🚢 Recommended Next Steps

### Phase 1: Core Deployment
1. Choose your cloud provider based on your needs
2. Follow the corresponding deployment guide
3. Deploy auth-service microservice
4. Verify service health and connectivity

### Phase 2: Enterprise Features
1. Configure **Ingress Controller** (Nginx, ALB, or cloud-native)
2. Set up **SSL/TLS certificates** (Let's Encrypt, ACM, Certbot)
3. Implement **DNS management**
4. Configure **auto-scaling policies**

### Phase 3: Observability
1. Deploy **Prometheus & Grafana**
2. Configure centralized **logging**
3. Set up **alerting** rules
4. Implement **SLA monitoring**

### Phase 4: Security & Compliance
1. Implement **network policies**
2. Set up **pod security policies**
3. Configure **RBAC** and access control
4. Regular **security audits**

### Phase 5: CI/CD Integration
1. Set up **GitOps** workflow (ArgoCD, Flux)
2. Automate **image builds** (Docker Hub, ECR, Artifact Registry, ACR)
3. Implement **automated deployments**
4. Configure **rollback** strategies

---

## 📞 Support & Resources

- **Kubernetes Official:** https://kubernetes.io/docs
- **AWS EKS:** https://docs.aws.amazon.com/eks
- **Google Cloud GKE:** https://cloud.google.com/kubernetes-engine/docs
- **Azure AKS:** https://docs.microsoft.com/en-us/azure/aks

---

## 📋 Version History

| Version | Date | Changes |
|---------|------|---------|
| 2.0.0 | 2025-11-13 | Multi-cloud deployment guides added |
| 1.0.1 | 2025-11-13 | Initial single-cloud documentation |

---

## 👥 Team & Contributions

This documentation is maintained by the ADES Infrastructure Team.

For contributions, improvements, or questions, please open an issue or contact the team.

---

**Status:** ✅ Production Ready | 🔄 Actively Maintained
