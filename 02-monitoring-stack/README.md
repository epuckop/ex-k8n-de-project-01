# Complete Kubernetes Monitoring Stack

This directory contains a comprehensive monitoring solution using the `kube-prometheus-stack` Helm chart, which includes Prometheus, Grafana, AlertManager, and all necessary exporters for complete Kubernetes cluster monitoring.

## Prerequisites

1. **ArgoCD installed and running**: Follow instructions in `../00-ArgoCD/README.md`
2. **ArgoCD CLI installed**: See `../00-ArgoCD/README.md` for installation instructions
3. **Infrastructure deployed**: Namespaces must exist (see `../01-manifests/README.md`)
4. **kubectl access**: Configured to access your Kubernetes cluster

## Components Included

### Core Monitoring Stack
- **Prometheus**: Metrics collection and storage with 30-day retention
- **Grafana**: Visualization and dashboards (replaces standalone Grafana)
- **AlertManager**: Alert routing and notification management
- **Prometheus Operator**: CRD-based Prometheus management

### Data Collection
- **Node Exporter**: System metrics from Kubernetes nodes
- **kube-state-metrics**: Kubernetes object state metrics
- **cAdvisor**: Container metrics (built into kubelet)

### Ready-to-Use Features
- **50+ Pre-built Dashboards**: Kubernetes cluster, nodes, pods, persistent volumes
- **100+ Default Alerts**: Critical system and application monitoring
- **ServiceMonitor/PodMonitor Support**: Automatic service discovery
- **Custom Resource Support**: Easy monitoring extension

## Installation Steps

### Step 1: Login to ArgoCD CLI
```bash
# Login to ArgoCD (see ../00-ArgoCD/README.md for setup)
argocd login localhost:30080 --insecure
```

### Step 2: Add Prometheus Community Helm Repository
```bash
argocd repo add https://prometheus-community.github.io/helm-charts --type helm --name prometheus-community
```

### Step 3: Deploy Infrastructure Dependencies
```bash
# Ensure namespaces exist (required first)
argocd app get infrastructure-manifests-namespace --refresh
```

### Step 4: Deploy Monitoring Stack

#### Local Development Configuration
```bash
argocd app create monitoring-stack \
  --repo https://github.com/epuckop/ex-k8n-de-project-01.git \
  --path 02-monitoring-stack \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace monitoring \
  --helm-set kube-prometheus-stack.grafana.adminPassword=YourSecurePassword123 \
  --helm-set kube-prometheus-stack.grafana.service.type=NodePort \
  --helm-set kube-prometheus-stack.grafana.service.nodePort=30300 \
  --helm-set kube-prometheus-stack.grafana.persistence.enabled=true \
  --helm-set kube-prometheus-stack.grafana.persistence.size=10Gi \
  --helm-set kube-prometheus-stack.prometheus.service.type=NodePort \
  --helm-set kube-prometheus-stack.prometheus.service.nodePort=30090 \
  --helm-set kube-prometheus-stack.prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi \
  --helm-set kube-prometheus-stack.alertmanager.service.type=NodePort \
  --helm-set kube-prometheus-stack.alertmanager.service.nodePort=30093 \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

#### Production Configuration
```bash
argocd app create monitoring-stack \
  --repo https://github.com/epuckop/ex-k8n-de-project-01.git \
  --path 02-monitoring-stack \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace monitoring \
  --helm-set kube-prometheus-stack.grafana.adminPassword=$GRAFANA_ADMIN_PASSWORD \
  --helm-set kube-prometheus-stack.grafana.persistence.enabled=true \
  --helm-set kube-prometheus-stack.grafana.persistence.size=20Gi \
  --helm-set kube-prometheus-stack.grafana.persistence.storageClassName=fast-ssd \
  --helm-set kube-prometheus-stack.prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=100Gi \
  --helm-set kube-prometheus-stack.prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=fast-ssd \
  --helm-set kube-prometheus-stack.alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.resources.requests.storage=10Gi \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

### Step 5: Wait for Deployment
```bash
# Wait for application to sync and become healthy
argocd app wait monitoring-stack --health

# Check deployment status
kubectl get pods -n monitoring
kubectl get pvc -n monitoring
```

## Access Services

### Grafana Dashboard
- **Local Development**: http://localhost:30300
- **Production (Port Forward)**: 
  ```bash
  kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
  # Access at http://localhost:3000
  ```
- **Credentials**: admin / (password set via ArgoCD parameter)

### Prometheus
- **Local Development**: http://localhost:30090  
- **Production (Port Forward)**:
  ```bash
  kubectl port-forward -n monitoring svc/monitoring-prometheus 9090:9090
  # Access at http://localhost:9090
  ```

### AlertManager
- **Local Development**: http://localhost:30093
- **Production (Port Forward)**:
  ```bash
  kubectl port-forward -n monitoring svc/monitoring-alertmanager 9093:9093
  # Access at http://localhost:9093
  ```

## Pre-built Dashboards

### Kubernetes Dashboards
- **Kubernetes / Compute Resources / Cluster**: Overall cluster resource usage
- **Kubernetes / Compute Resources / Namespace (Pods)**: Per-namespace pod resources
- **Kubernetes / Compute Resources / Node (Pods)**: Per-node pod resources
- **Kubernetes / Networking / Cluster**: Network traffic and DNS
- **Kubernetes / Persistent Volumes**: Storage usage and performance

### Node and System Dashboards  
- **Node Exporter / Nodes**: Detailed node system metrics
- **Node Exporter / USE Method / Node**: Node utilization, saturation, errors
- **Node Exporter / USE Method / Cluster**: Cluster-wide system metrics

### Prometheus Stack Dashboards
- **Prometheus / Overview**: Prometheus server health and performance
- **AlertManager / Overview**: Alert routing and notification status

## Default Monitoring Targets

The stack automatically monitors:

### Kubernetes Control Plane
- **API Server**: Request rates, latencies, authentication
- **etcd**: Cluster store health and performance  
- **Scheduler**: Pod scheduling metrics
- **Controller Manager**: Workload controller health

### Kubernetes Workloads
- **Pods**: CPU, memory, network, filesystem usage
- **Deployments**: Replica availability and rollout status
- **Services**: Endpoint availability and traffic
- **Ingress**: HTTP request metrics (if ingress controller supports metrics)

### System Resources
- **Nodes**: CPU, memory, disk, network utilization
- **Container Runtime**: Docker/containerd metrics
- **Network**: CNI plugin metrics (if supported)

## Configuration Management

### Required ArgoCD Parameters
- `kube-prometheus-stack.grafana.adminPassword` - **REQUIRED** - Grafana admin password

### Storage Parameters
- `kube-prometheus-stack.grafana.persistence.enabled=true` - Enable Grafana persistence
- `kube-prometheus-stack.grafana.persistence.size=10Gi` - Grafana storage size
- `kube-prometheus-stack.prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi` - Prometheus storage

### Service Access Parameters
- `kube-prometheus-stack.grafana.service.type=NodePort` - Grafana service type
- `kube-prometheus-stack.grafana.service.nodePort=30300` - Grafana NodePort
- `kube-prometheus-stack.prometheus.service.type=NodePort` - Prometheus service type
- `kube-prometheus-stack.alertmanager.service.type=NodePort` - AlertManager service type

## Extending Monitoring

### Monitor Custom Applications

Create ServiceMonitor for your application:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: metrics
    path: /metrics
    interval: 30s
```

### Custom Dashboards
Place JSON dashboard files in:
- Repository: Create `02-monitoring-stack/dashboards/`
- Runtime: Import via Grafana UI

### Custom Alerts
Create PrometheusRule resources:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-app-rules
  namespace: monitoring
spec:
  groups:
  - name: my-app.rules
    rules:
    - alert: MyAppDown
      expr: up{job="my-app"} == 0
      for: 5m
      annotations:
        summary: "My App is down"
```

## ArgoCD Management Commands

### Application Management
```bash
# View application status
argocd app get monitoring-stack

# Manual sync
argocd app sync monitoring-stack

# Update parameters
argocd app set monitoring-stack \
  --helm-set kube-prometheus-stack.grafana.persistence.size=20Gi

# View sync history
argocd app history monitoring-stack

# Rollback if needed
argocd app rollback monitoring-stack <REVISION_ID>
```

### Configuration Updates
```bash
# Method 1: Update Helm parameters (Recommended)
argocd app set monitoring-stack \
  --helm-set kube-prometheus-stack.prometheus.prometheusSpec.retention=60d \
  --helm-set kube-prometheus-stack.grafana.plugins[0]=grafana-piechart-panel

# Method 2: Edit application manifest
argocd app edit monitoring-stack

# Method 3: Template updates (commit to Git and auto-sync)
```

## Troubleshooting

### Common Issues

#### Application Not Syncing
```bash
# Check application health
argocd app get monitoring-stack

# Check repository access
argocd repo list

# Force refresh
argocd app get monitoring-stack --refresh
```

#### Pods Not Starting
```bash
# Check pod status and events
kubectl describe pods -n monitoring
kubectl get events -n monitoring --sort-by='.lastTimestamp'

# Check persistent volume claims
kubectl get pvc -n monitoring
kubectl describe pvc -n monitoring
```

#### High Resource Usage
```bash
# Check resource usage
kubectl top pods -n monitoring
kubectl top nodes

# Scale down if needed (temporarily)
kubectl scale deployment monitoring-kube-state-metrics -n monitoring --replicas=0
```

#### Storage Issues
```bash
# Check PVC status
kubectl get pvc -n monitoring

# Check storage class
kubectl get storageclass

# Check node disk space
kubectl describe nodes
```

### Reset Admin Password
```bash
# Delete Grafana admin secret to regenerate
kubectl delete secret monitoring-grafana -n monitoring

# Restart Grafana pod
kubectl delete pod -l app.kubernetes.io/name=grafana -n monitoring
```

## Performance Tuning

### Prometheus Optimization
- **Retention**: Adjust based on storage capacity
- **Scrape Interval**: Balance between data resolution and resource usage
- **Recording Rules**: Pre-compute expensive queries

### Resource Requests/Limits
- **Prometheus**: Scales with number of metrics and retention
- **Grafana**: Relatively lightweight
- **AlertManager**: Minimal resources needed

## Security Considerations

### Network Security
- Services use ClusterIP by default (internal access only)
- NodePort services should be restricted in production
- Consider NetworkPolicies for traffic isolation

### RBAC
- Prometheus has cluster-wide read permissions (required)
- Grafana uses service account with minimal permissions
- ServiceMonitors require proper RBAC for target services

### Data Protection
- Enable persistence for production environments  
- Consider backup strategies for Prometheus data
- Secure Grafana admin credentials via ArgoCD parameters

## Version Information

- **kube-prometheus-stack**: v77.6.0
- **Prometheus**: v2.52.x
- **Grafana**: v11.x.x  
- **AlertManager**: v0.27.x
- **Kubernetes**: 1.25+ required

## Files in this Directory

- `README.md` - This documentation
- `Chart.yaml` - Helm chart with kube-prometheus-stack dependency  
- `values.yaml` - Complete monitoring stack configuration