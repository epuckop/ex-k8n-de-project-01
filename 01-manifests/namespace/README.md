# Infrastructure Manifests - Namespaces

This directory contains Kubernetes namespace definitions for the monitoring stack.

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
argocd app create monitoring-namespace \
  --repo https://github.com/epuckop/ex-k8n-de-project-01.git \
  --path 01-manifests/namespace \
  --dest-server https://kubernetes.default.svc \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

## Dependencies

This namespace should be created **before** any monitoring applications:
1. `00-ArgoCD` - ArgoCD installation
2. `01-manifests/namespace` - Monitoring namespace (this)
3. `02-grafana-stack` - Grafana deployment
4. `03-prometheus-stack` - Prometheus deployment (future)
5. Other monitoring applications

## GitOps Benefits

- **Infrastructure as Code**: Namespace definition in Git
- **Version Control**: Track changes to namespace configuration
- **Automated Recovery**: ArgoCD will recreate if namespace is deleted
- **Audit Trail**: All changes tracked in Git history