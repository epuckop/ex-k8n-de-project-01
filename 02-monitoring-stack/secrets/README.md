# Monitoring Secrets Management

This directory contains Helm chart for managing secrets and configuration for the monitoring stack.

## What This Creates

### Kubernetes Secret: `monitoring-secrets`
Contains sensitive data for monitoring components:
- `grafana-admin-user`: Grafana admin username (default: admin)
- `grafana-admin-password`: Grafana admin password (set via ArgoCD parameter)

### Kubernetes ConfigMap: `monitoring-config`  
Contains non-sensitive configuration:
- Storage sizes for Grafana, Prometheus, AlertManager
- Retention policies
- Service types and NodePorts
- Resource requests and limits

## Deployment Options

### Option 1: Via ArgoCD Application (Recommended)

```bash
# Deploy secrets first (before monitoring stack)
argocd app create monitoring-secrets \
  --repo https://github.com/epuckop/ex-k8n-de-project-01.git \
  --path 02-monitoring-stack/secrets \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace monitoring \
  --helm-set grafana.admin.password=YourSecurePassword123 \
  --helm-set service.type=NodePort \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

### Option 2: Manual kubectl (Alternative)

```bash
# Use the helper script
./scripts/create-secrets.sh monitoring YourSecurePassword123

# Or manually
kubectl create secret generic monitoring-secrets \
  --from-literal=grafana-admin-user=admin \
  --from-literal=grafana-admin-password=YourSecurePassword123 \
  -n monitoring

kubectl create configmap monitoring-config \
  --from-literal=grafana-storage-size=10Gi \
  --from-literal=prometheus-storage-size=50Gi \
  -n monitoring
```

## Configuration Parameters

### Required Parameters
- `grafana.admin.password` - **REQUIRED** - Grafana admin password

### Storage Parameters
- `storage.grafana.size=10Gi` - Grafana persistent volume size
- `storage.prometheus.size=50Gi` - Prometheus storage size
- `storage.alertmanager.size=10Gi` - AlertManager storage size

### Service Parameters  
- `service.type=NodePort|ClusterIP` - Service type (NodePort for dev, ClusterIP for prod)
- `service.grafana.nodePort=30300` - Grafana NodePort
- `service.prometheus.nodePort=30090` - Prometheus NodePort
- `service.alertmanager.nodePort=30093` - AlertManager NodePort

## Editing Secrets After Deployment

### Via kubectl
```bash
# Edit secret interactively
kubectl edit secret monitoring-secrets -n monitoring

# Update specific value
kubectl patch secret monitoring-secrets -n monitoring \
  --patch='{"data":{"grafana-admin-password":"TmV3UGFzc3dvcmQ="}}'

# Update configmap
kubectl patch configmap monitoring-config -n monitoring \
  --patch='{"data":{"prometheus-storage-size":"100Gi"}}'
```

### Via ArgoCD Parameters
```bash
# Update application parameters
argocd app set monitoring-secrets \
  --helm-set grafana.admin.password=NewPassword123 \
  --helm-set storage.prometheus.size=100Gi

# Sync changes
argocd app sync monitoring-secrets
```

### Via UI Tools
- **k9s**: Navigate to secrets/configmaps and edit interactively
- **Lens**: Use built-in editor for Kubernetes resources
- **Kubernetes Dashboard**: Web UI for resource editing

## Verification

```bash
# Check created resources
kubectl get secrets,configmaps -n monitoring

# View secret data (base64 decoded)
kubectl get secret monitoring-secrets -n monitoring -o yaml

# View configmap data
kubectl get configmap monitoring-config -n monitoring -o yaml

# Check ArgoCD sync status
argocd app get monitoring-secrets
```

## Security Notes

- Secrets are stored in Kubernetes etcd (encrypted at rest if configured)
- Use strong passwords for Grafana admin account
- Consider rotating passwords periodically
- Monitor ArgoCD audit logs for parameter changes
- Secrets are base64 encoded (not encrypted) in Kubernetes

## Troubleshooting

### Secret Creation Failed
```bash
# Check namespace exists
kubectl get namespace monitoring

# Check RBAC permissions
kubectl auth can-i create secrets --namespace monitoring

# View ArgoCD application events
argocd app get monitoring-secrets --show-params
```

### Values Not Applied
```bash
# Force refresh ArgoCD application
argocd app get monitoring-secrets --refresh

# Check Helm values resolution
kubectl get secret monitoring-secrets -n monitoring -o jsonpath='{.data.grafana-admin-password}' | base64 -d
```

## Files in this Directory

- `Chart.yaml` - Helm chart metadata with sync-wave annotation
- `values.yaml` - Default values and parameter documentation
- `templates/secrets.yaml` - Secret resource template
- `templates/configmaps.yaml` - ConfigMap resource template
- `templates/_helpers.tpl` - Helm template helpers
- `README.md` - This documentation