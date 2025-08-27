# Grafana Stack Deployment with ArgoCD

This directory contains Grafana monitoring stack configuration for deployment via ArgoCD using Helm.

## Prerequisites

1. **ArgoCD installed and running**: Follow instructions in `../00-ArgoCD/README.md`
2. **ArgoCD CLI installed**: 
   ```bash
   # macOS
   brew install argocd
   
   # Linux
   curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
   sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
   ```

3. **kubectl access**: Configured to access your Kubernetes cluster

## Architecture

- **Chart.yaml**: Helm chart with dependency on official Grafana chart
- **values.yaml**: Configuration for local development with NodePort access
- **Deployment**: via ArgoCD CLI managing Helm application

## Installation Steps

### Step 1: Login to ArgoCD CLI
```bash
# Login to ArgoCD (use your actual ArgoCD server URL)
argocd login localhost:30080 --insecure

# Enter credentials:
# Username: admin
# Password: (get from kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
```

### Step 2: Add Grafana Helm Repository to ArgoCD
```bash
argocd repo add https://grafana.github.io/helm-charts --type helm --name grafana-charts
```

### Step 3: Add Your Git Repository to ArgoCD
```bash
# Replace with your actual repository URL
argocd repo add https://github.com/USERNAME/REPOSITORY.git --name my-project
```

### Step 4: Create Monitoring Namespace via ArgoCD
```bash
# Create namespace application first
argocd app create monitoring-namespace \
  --repo https://github.com/USERNAME/REPOSITORY.git \
  --path manifests/namespace \
  --dest-server https://kubernetes.default.svc \
  --sync-policy automated
```

### Step 5: Create ArgoCD Application for Grafana with Parameters
```bash
# Basic deployment with minimal configuration
argocd app create grafana-stack \
  --repo https://github.com/USERNAME/REPOSITORY.git \
  --path 01-grafana-stack \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace monitoring \
  --helm-set grafana.adminPassword=YourSecurePassword123 \
  --helm-set grafana.persistence.enabled=true \
  --helm-set grafana.persistence.size=10Gi \
  --helm-set grafana.service.type=NodePort \
  --helm-set grafana.service.nodePort=30300 \
  --sync-policy automated \
  --auto-prune \
  --self-heal

# Production deployment with additional parameters
argocd app create grafana-stack \
  --repo https://github.com/USERNAME/REPOSITORY.git \
  --path 01-grafana-stack \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace monitoring \
  --helm-set grafana.adminPassword=$GRAFANA_ADMIN_PASSWORD \
  --helm-set grafana.persistence.enabled=true \
  --helm-set grafana.persistence.size=50Gi \
  --helm-set grafana.persistence.storageClassName=fast-ssd \
  --helm-set grafana.service.type=ClusterIP \
  --helm-set grafana.resources.requests.cpu=500m \
  --helm-set grafana.resources.requests.memory=512Mi \
  --helm-set grafana.resources.limits.cpu=1000m \
  --helm-set grafana.resources.limits.memory=1Gi \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

### Step 6: Sync the Application
```bash
# Manual sync (if auto-sync not enabled)
argocd app sync grafana-stack

# Check sync status
argocd app get grafana-stack
```

## Access Grafana

### Local Development Access (with NodePort)
- **URL**: http://localhost:30300
- **Username**: admin  
- **Password**: The password you set via `--helm-set grafana.adminPassword=`

### Production Access (with ClusterIP)
```bash
# Port-forward for secure access
kubectl port-forward -n monitoring svc/grafana-stack-grafana 3000:80

# Access at http://localhost:3000
```

### Verify Deployment
```bash
# Check namespace and pods
kubectl get pods -n monitoring
kubectl get svc -n monitoring

# Check ArgoCD application status
argocd app list
argocd app get grafana-stack
```

## Configuration Details

### Grafana Configuration (values.yaml)
- **Template-based**: No sensitive data in repository
- **Configurable via ArgoCD parameters**: Service type, storage, credentials
- **Default Plugins**: Piechart, Worldmap, Clock panels
- **Secure by default**: Admin credentials set only via parameters
- **TestData Datasource**: Pre-configured for testing

### Required ArgoCD Parameters
- `grafana.adminPassword` - **REQUIRED** - Admin password
- `grafana.service.type` - Service type (NodePort/ClusterIP)
- `grafana.persistence.enabled` - Enable persistent storage
- `grafana.persistence.size` - Storage size (e.g., 10Gi)

### Optional ArgoCD Parameters
- `grafana.service.nodePort` - NodePort number (30000-32767)
- `grafana.persistence.storageClassName` - Storage class
- `grafana.resources.requests.cpu/memory` - Resource requests
- `grafana.resources.limits.cpu/memory` - Resource limits

### Helm Chart Structure
- **Base Chart**: grafana/grafana v8.7.1
- **Grafana Version**: 11.3.1
- **Dependencies**: Official Grafana Helm chart

## ArgoCD Management Commands

### Application Management
```bash
# View application details
argocd app get grafana-stack

# Manual sync
argocd app sync grafana-stack

# View sync history
argocd app history grafana-stack

# Rollback to previous version
argocd app rollback grafana-stack <REVISION_ID>

# Delete application
argocd app delete grafana-stack
```

### Repository Management
```bash
# List repositories
argocd repo list

# Remove repository
argocd repo rm https://github.com/USERNAME/REPOSITORY.git
```

## Updating Configuration

### Method 1: Update Helm Parameters (Recommended)
```bash
# Update application with new parameters
argocd app set grafana-stack \
  --helm-set grafana.persistence.size=20Gi \
  --helm-set grafana.service.nodePort=30301

# Sync changes
argocd app sync grafana-stack
```

### Method 2: Update via Application Manifest
```bash
# Edit application in ArgoCD
argocd app edit grafana-stack

# Or patch specific parameters
argocd app patch grafana-stack --patch '{"spec":{"source":{"helm":{"parameters":[{"name":"grafana.persistence.size","value":"20Gi"}]}}}}'
```

### Method 3: Template Updates (Non-sensitive)
1. **Modify values.yaml** template in this directory
2. **Commit changes** to Git repository  
3. **ArgoCD will auto-sync** (if auto-sync enabled)
4. **Or manually sync**: `argocd app sync grafana-stack`

## Troubleshooting

### Common Issues

#### Application Not Syncing
```bash
# Check application status
argocd app get grafana-stack

# Check repository connection
argocd repo list

# Force refresh repository
argocd app get grafana-stack --refresh
```

#### Pods Not Starting
```bash
# Check pod status and events
kubectl describe pods -n monitoring
kubectl get events -n monitoring --sort-by='.lastTimestamp'

# Check persistent volume claims
kubectl get pvc -n monitoring
```

#### Cannot Access Grafana UI
```bash
# Check service status
kubectl get svc -n monitoring

# Check if NodePort is accessible
curl http://localhost:30300
```

#### Wrong Grafana Version
```bash
# Check running version
kubectl get pods -n monitoring -o=jsonpath='{.items[0].spec.containers[0].image}'

# Update Chart.yaml or values.yaml and commit changes
```

### Reset Grafana Admin Password
```bash
# Delete admin password secret to regenerate
kubectl delete secret grafana-stack-grafana-admin -n monitoring

# Restart Grafana pod
kubectl delete pod -l app.kubernetes.io/name=grafana -n monitoring
```

## Monitoring Integration

This Grafana stack is ready for integration with:
- **Prometheus**: Add datasource configuration in values.yaml
- **Loki**: For log aggregation
- **AlertManager**: For alert management

### Adding Prometheus Datasource
Add to `values.yaml`:
```yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-server.monitoring.svc.cluster.local
      access: proxy
      isDefault: true
```

## Files in this Directory

- `README.md` - This documentation
- `Chart.yaml` - Helm chart definition with Grafana dependency
- `values.yaml` - Grafana configuration for local development

## Version Information

- **Grafana Version**: 11.3.1
- **Helm Chart Version**: 8.7.1
- **Kubernetes Version**: 1.25+
- **ArgoCD Version**: Compatible with v3.1.1+

## Additional Resources

- [Grafana Helm Chart Documentation](https://grafana.github.io/helm-charts)
- [ArgoCD CLI Documentation](https://argo-cd.readthedocs.io/en/stable/cli_installation/)
- [Grafana Configuration Reference](https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/)