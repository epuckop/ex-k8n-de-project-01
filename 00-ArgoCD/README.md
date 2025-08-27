# ArgoCD Deployment with Helm

This directory contains the configuration and instructions for deploying ArgoCD using Helm.

## Prerequisites

Before deploying ArgoCD, ensure you have the following dependencies installed:

### Required Dependencies

1. **Kubernetes Cluster**: A running Kubernetes cluster (v1.21+ recommended)
2. **kubectl**: Kubernetes command-line tool configured to access your cluster
3. **Helm**: Helm package manager (v3.0+ required)

### Installation Commands for Dependencies

#### Install kubectl
```bash
# macOS (using Homebrew)
brew install kubectl

# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Windows (using Chocolatey)
choco install kubernetes-cli
```

#### Install Helm
```bash
# macOS (using Homebrew)
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Windows (using Chocolatey)
choco install kubernetes-helm
```

### Verify Dependencies
```bash
# Check Kubernetes connection
kubectl cluster-info

# Check Helm version
helm version

# Check kubectl version
kubectl version --client
```

## Installation

### Step 1: Add ArgoCD Helm Repository
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

### Step 2: Deploy ArgoCD

#### Option A: Full deployment with all features
```bash
helm install argocd argo/argo-cd \
  --version 8.3.1 \
  --namespace argocd \
  --create-namespace \
  --values values.yaml
```

#### Option B: Minimal local deployment (Recommended for development)
```bash
helm install argocd argo/argo-cd \
  --version 8.3.1 \
  --namespace argocd \
  --create-namespace \
  --values values-local.yaml
```

### Step 3: Wait for Deployment
```bash
# Wait for all pods to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s

# Check deployment status
kubectl get pods -n argocd
```

### Step 4: Get Initial Admin Password
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

### Step 5: Access ArgoCD UI

#### Option A: Port Forward (if using values.yaml)
```bash
kubectl port-forward service/argocd-server -n argocd 8080:443
```
Access at: https://localhost:8080

#### Option B: Direct access (if using values-local.yaml with NodePort)
Access directly at: http://localhost:30080
No port-forwarding needed!

#### Option C: LoadBalancer (if configured)
```bash
kubectl get svc argocd-server -n argocd
```

## Login Credentials

- **Username**: `admin`
- **Password**: Retrieved from Step 4 above

## Configuration

Two configuration options are available:

- **`values.yaml`**: Full official ArgoCD values file with all configuration options
- **`values-local.yaml`**: Minimal configuration for local development with:
  - Insecure mode (no TLS)
  - NodePort service (permanent access on localhost:30080)
  - Embedded Redis (no HA)
  - Disabled Dex, notifications, ApplicationSet
  - Simple admin RBAC

## Verification

### Check ArgoCD Status
```bash
# Check all ArgoCD pods
kubectl get pods -n argocd

# Check ArgoCD services
kubectl get svc -n argocd

# Check ArgoCD server logs
kubectl logs -n argocd deployment/argocd-server
```

### Test ArgoCD CLI (Optional)
```bash
# Install ArgoCD CLI
brew install argocd  # macOS
# or download from https://github.com/argoproj/argo-cd/releases

# Login via CLI
argocd login localhost:8080 --insecure
```

## Uninstallation

### Step 1: Delete ArgoCD Applications (if any)
```bash
# List all applications
kubectl get applications -n argocd

# Delete applications (replace APP_NAME with actual names)
kubectl delete application APP_NAME -n argocd
```

### Step 2: Uninstall Helm Release
```bash
helm uninstall argocd --namespace argocd
```

### Step 3: Delete Namespace (Complete Cleanup)
```bash
# Warning: This will delete everything in the argocd namespace
kubectl delete namespace argocd
```

### Step 4: Remove Helm Repository (Optional)
```bash
helm repo remove argo
```

## Troubleshooting

### Common Issues

#### Pods not starting
```bash
# Check pod status and events
kubectl describe pods -n argocd
kubectl get events -n argocd --sort-by='.lastTimestamp'
```

#### Cannot access UI
```bash
# Check service status
kubectl get svc argocd-server -n argocd

# Check if port-forward is working
kubectl port-forward service/argocd-server -n argocd 8080:443 --address 0.0.0.0
```

#### Forgot admin password
```bash
# Reset admin password
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {"admin.password": "$2a$10$rRyBsGSHK6.uc8fntPwVIuLVHgsAhAX7TcdrqW/RADU0uh7CaChLa","admin.passwordMtime": "'$(date +%FT%T%Z)'"}}'

# New password is: password
```

## Files in this Directory

- `README.md` - This documentation
- `values.yaml` - Full official ArgoCD Helm values file for production
- `values-local.yaml` - Minimal configuration for local development

## Version Information

- **ArgoCD Version**: v3.1.1
- **Helm Chart Version**: 8.3.1
- **Kubernetes Version**: 1.25+

## Additional Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [ArgoCD Helm Chart](https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd)
- [Kubernetes Documentation](https://kubernetes.io/docs/)