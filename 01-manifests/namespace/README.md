# Infrastructure Manifests

This directory contains core Kubernetes infrastructure manifests that should be deployed before applications.

## Current Structure

- `namespace/` - Namespace definitions for all application environments

## Files

- `namespace.yaml` - All namespace definitions (monitoring, frontend, backend)

## Namespaces

### monitoring
Namespace for monitoring-related applications:
- Grafana
- Prometheus  
- AlertManager
- Other monitoring tools

**Labels:**
- `name: monitoring`
- `purpose: monitoring-stack`

### frontend
Namespace for frontend applications:
- Flutter web applications
- Static web content
- Frontend services

**Labels:**
- `name: frontend`
- `purpose: frontend-applications`

### backend  
Namespace for backend applications:
- Python APIs
- Database services
- Backend microservices

**Labels:**
- `name: backend`
- `purpose: backend-applications`

### Common Annotations:
- `argocd.argoproj.io/sync-wave: "-1"` - Ensures namespaces are created before applications

## ArgoCD Deployment

Deploy this namespace via ArgoCD:

```bash
argocd app create infrastructure-manifests-namespace \
  --repo https://github.com/epuckop/ex-k8n-de-project-01.git \
  --path 01-manifests/namespace \
  --dest-server https://kubernetes.default.svc \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

## Dependencies

This infrastructure should be created **before** any applications:
1. `00-ArgoCD` - ArgoCD installation
2. `01-manifests` - Infrastructure manifests (this) - Creates all namespaces
3. `02-grafana-stack` - Grafana deployment (→ monitoring namespace)
4. `03-prometheus-stack` - Prometheus deployment (→ monitoring namespace)
5. `04-frontend-app` - Frontend applications (→ frontend namespace)
6. `05-backend-app` - Backend applications (→ backend namespace)

## GitOps Benefits

- **Infrastructure as Code**: Namespace definition in Git
- **Version Control**: Track changes to namespace configuration
- **Automated Recovery**: ArgoCD will recreate if namespace is deleted
- **Audit Trail**: All changes tracked in Git history