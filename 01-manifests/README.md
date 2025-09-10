# Infrastructure Manifests

This directory contains Kubernetes infrastructure manifests that are deployed via ArgoCD applications.

## Prerequisites

### Required: ArgoCD Installation and Authentication

Before deploying any manifests from this directory, you **must** have ArgoCD installed and properly authenticated.

#### 1. Install ArgoCD
Follow the complete installation guide in `../00-ArgoCD/README.md`:

```bash
# Quick installation for local development
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  --version 8.3.1 \
  --namespace argocd \
  --create-namespace \
  --values ../00-ArgoCD/values-local.yaml
```

#### 2. Setup ArgoCD Authentication

**Get Initial Admin Password:**
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

**Login via ArgoCD CLI:**
```bash
# Install ArgoCD CLI (see ../00-ArgoCD/README.md for installation instructions)

# Login to ArgoCD
argocd login localhost:30080 --insecure
# Username: admin
# Password: (from command above)
```

**Add Repository to ArgoCD:**
```bash
# Add this Git repository to ArgoCD
argocd repo add https://github.com/epuckop/ex-k8n-de-project-01.git --name ex-k8n-de-project-01
```

## Namespace Structure

### `namespace/namespace.yaml`
Defines three core namespaces for application segregation:

- **`monitoring`**: For monitoring stack (Grafana, Prometheus, etc.)
- **`frontend`**: For frontend applications and services  
- **`backend`**: For backend applications and APIs

### Sync Wave Priority
All namespaces use `argocd.argoproj.io/sync-wave: "-1"` to ensure they are created **before** any applications that depend on them.

## Deployment via ArgoCD

### Method 1: Create ArgoCD Application (Recommended)

**Deploy all namespaces:**
```bash
argocd app create infrastructure-manifests-namespace \
  --repo https://github.com/epuckop/ex-k8n-de-project-01.git \
  --path 01-manifests/namespace \
  --dest-server https://kubernetes.default.svc \
  --sync-policy automated \
  --auto-prune \
  --self-heal

# Wait for deployment to complete
argocd app wait infrastructure-manifests-namespace --health
```

**Verify deployment:**
```bash
# Check ArgoCD application status
argocd app get infrastructure-manifests-namespace

# Verify namespaces were created
kubectl get namespaces monitoring frontend backend
```

### Method 2: Manual kubectl (Not Recommended)

Only use if ArgoCD is not available:
```bash
kubectl apply -f namespace/namespace.yaml
```

## ArgoCD Authorization Configuration

### Repository Access
ArgoCD needs read access to this Git repository:

```bash
# Public repository (no authentication needed)
argocd repo add https://github.com/USERNAME/REPOSITORY.git

# Private repository (requires authentication)
argocd repo add https://github.com/USERNAME/REPOSITORY.git \
  --username <USERNAME> \
  --password <PERSONAL_ACCESS_TOKEN>
```

### Cluster Permissions
ArgoCD requires cluster-admin permissions to manage namespaces and applications:

```bash
# Verify ArgoCD has proper permissions
kubectl auth can-i create namespaces --as=system:serviceaccount:argocd:argocd-application-controller

# Should return "yes"
```

### RBAC Configuration
The ArgoCD installation includes appropriate RBAC policies:

- **Application Controller**: Can manage all Kubernetes resources
- **Server**: Can read/write ArgoCD applications and projects
- **Repo Server**: Can access Git repositories and Helm charts

## Usage in Other Applications

Once namespaces are deployed, other ArgoCD applications can reference them:

```bash
# Example: Deploy monitoring stack to monitoring namespace
argocd app create monitoring-stack \
  --repo https://github.com/epuckop/ex-k8n-de-project-01.git \
  --path 02-monitoring-stack \
  --dest-namespace monitoring \
  --dest-server https://kubernetes.default.svc
```

## Troubleshooting

### Application Not Syncing
```bash
# Check repository connection
argocd repo list

# Verify repository access
argocd repo get https://github.com/epuckop/ex-k8n-de-project-01.git

# Force application refresh
argocd app get infrastructure-manifests-namespace --refresh
```

### Permission Denied Errors
```bash
# Check ArgoCD service account permissions
kubectl describe clusterrolebinding argocd-application-controller

# Verify ArgoCD pods are running
kubectl get pods -n argocd
```

### Namespace Creation Failed
```bash
# Check events for errors
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Check ArgoCD application controller logs
kubectl logs -n argocd deployment/argocd-application-controller
```

## Dependencies

This infrastructure must be deployed **before** any applications that require these namespaces:

1. **Deploy First**: `01-manifests/namespace` (this directory)
2. **Deploy After**: `02-monitoring-stack`, future frontend/backend applications

The sync wave ordering ensures proper dependency management when using ArgoCD automated sync.

## Security Considerations

- **Namespace Isolation**: Applications are segregated by purpose and access patterns
- **RBAC Policies**: Each namespace can have specific RBAC configurations
- **Network Policies**: Can be applied per namespace for traffic isolation
- **Resource Quotas**: Can be configured per namespace for resource management

## Files in this Directory

- `README.md` - This documentation
- `namespace/namespace.yaml` - Core namespace definitions with sync wave ordering