# Infrastructure Manifests

This directory contains core Kubernetes infrastructure manifests that should be deployed before applications.

## Current Structure

- `namespace/` - Namespace definitions (monitoring, logging, etc.)

## Files

- `namespace.yaml` - Monitoring namespace definition

## Namespace: monitoring

The `monitoring` namespace is created for all monitoring-related applications including:
- Grafana
- Prometheus  
- AlertManager
- Other monitoring tools

### Labels:
- `name: monitoring` - Standard namespace label
- `purpose: monitoring-stack` - Identifies this as monitoring infrastructure

### Annotations:
- `argocd.argoproj.io/sync-wave: "-1"` - Ensures namespace is created before applications

## ArgoCD Deployment

Deploy this namespace via ArgoCD:

```bash
argocd app create infrastructure-manifests \
  --repo https://github.com/epuckop/ex-k8n-de-project-01.git \
  --path 01-manifests \
  --dest-server https://kubernetes.default.svc \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

## Dependencies

This infrastructure should be created **before** any applications:
1. `00-ArgoCD` - ArgoCD installation
2. `01-manifests` - Infrastructure manifests (this)
3. `02-grafana-stack` - Grafana deployment
4. `03-prometheus-stack` - Prometheus deployment (future)
5. Other applications

## GitOps Benefits

- **Infrastructure as Code**: Namespace definition in Git
- **Version Control**: Track changes to namespace configuration
- **Automated Recovery**: ArgoCD will recreate if namespace is deleted
- **Audit Trail**: All changes tracked in Git history