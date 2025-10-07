# Monitoring Setup for hello-world-gke

This directory contains Kubernetes manifests for deploying the kube-prometheus monitoring stack to your GKE cluster.

## Overview

The kube-prometheus stack provides comprehensive monitoring for Kubernetes clusters using:
- **Prometheus Operator** - Manages Prometheus instances
- **Prometheus** - Metrics collection and alerting
- **Alertmanager** - Alert handling and routing
- **Grafana** - Visualization and dashboards
- **Node Exporter** - Hardware and OS metrics
- **Kube State Metrics** - Kubernetes object metrics
- **Prometheus Adapter** - Custom metrics API

## Directory Structure

This monitoring folder should contain:
```
monitoring/
├── README.md (this file)
├── setup/
│   ├── namespace.yaml
│   ├── 0alertmanagerConfigCustomResourceDefinition.yaml
│   ├── 0alertmanagerCustomResourceDefinition.yaml
│   ├── 0podmonitorCustomResourceDefinition.yaml
│   ├── 0probeCustomResourceDefinition.yaml
│   ├── 0prometheusCustomResourceDefinition.yaml
│   ├── 0prometheusagentCustomResourceDefinition.yaml
│   ├── 0prometheusruleCustomResourceDefinition.yaml
│   ├── 0scrapeconfigCustomResourceDefinition.yaml
│   ├── 0servicemonitorCustomResourceDefinition.yaml
│   └── 0thanosrulerCustomResourceDefinition.yaml
└── manifests/
    └── (all other manifest YAML files)
```

## Getting the Manifest Files

The manifest files should be copied from the [kube-prometheus repository](https://github.com/prometheus-operator/kube-prometheus). To populate this directory:

### Option 1: Clone and Copy (Recommended)

```bash
# Clone the kube-prometheus repository
git clone https://github.com/prometheus-operator/kube-prometheus.git /tmp/kube-prometheus
cd /tmp/kube-prometheus

# Checkout a stable release (recommended)
git checkout release-0.16

# Navigate to your hello-world-gke repository
cd /path/to/hello-world-gke

# Copy the setup manifests
cp -r /tmp/kube-prometheus/manifests/setup monitoring/

# Copy the main manifests
mkdir -p monitoring/manifests
cp /tmp/kube-prometheus/manifests/*.yaml monitoring/manifests/

# Clean up
rm -rf /tmp/kube-prometheus
```

### Option 2: Direct Download

Alternatively, you can download the manifests directly from GitHub:

```bash
# Download setup manifests
mkdir -p monitoring/setup monitoring/manifests
curl -L https://github.com/prometheus-operator/kube-prometheus/archive/refs/heads/main.tar.gz | tar xz --strip=3 -C monitoring/setup kube-prometheus-main/manifests/setup
curl -L https://github.com/prometheus-operator/kube-prometheus/archive/refs/heads/main.tar.gz | tar xz --strip=2 -C monitoring/manifests kube-prometheus-main/manifests --exclude='*/setup'
```

## Prerequisites

1. **GKE Cluster**: A running GKE cluster provisioned via this repository's terraform configuration
2. **kubectl**: Configured to communicate with your GKE cluster
3. **Cluster Access**: Appropriate RBAC permissions to create cluster-level resources

### GKE-Specific Requirements

For GKE, ensure your cluster has:
- Kubernetes version 1.31 or newer (for compatibility with release-0.16)
- At least 6GB of available memory across nodes
- Workload Identity enabled (optional, for enhanced security)

## Deployment Instructions

### Step 1: Connect to Your GKE Cluster

```bash
# Authenticate with Google Cloud
gcloud auth login

# Set your project
gcloud config set project YOUR_PROJECT_ID

# Get cluster credentials
gcloud container clusters get-credentials YOUR_CLUSTER_NAME --zone YOUR_ZONE

# Verify connection
kubectl cluster-info
```

### Step 2: Deploy CRDs and Namespace

First, create the monitoring namespace and Custom Resource Definitions (CRDs). This must be done before deploying the main components:

```bash
# Apply the setup manifests (creates namespace and CRDs)
kubectl apply --server-side -f monitoring/setup

# Wait for CRDs to be established
kubectl wait \
    --for condition=Established \
    --all CustomResourceDefinition \
    --namespace=monitoring \
    --timeout=120s
```

**Note**: We use `--server-side` apply because some CRDs are large and may exceed client-side limits.

### Step 3: Deploy Monitoring Stack

Once CRDs are established, deploy the main monitoring components:

```bash
# Apply all monitoring manifests
kubectl apply -f monitoring/manifests/
```

If you encounter any errors, you may need to run the command again as some resources may have dependencies.

### Step 4: Verify Deployment

Check that all pods are running:

```bash
# Watch pods come up
kubectl get pods -n monitoring -w

# Check all resources
kubectl get all -n monitoring
```

You should see pods for:
- `prometheus-operator`
- `prometheus-k8s` (2 replicas)
- `alertmanager-main` (3 replicas)
- `grafana`
- `kube-state-metrics`
- `node-exporter` (DaemonSet on each node)
- `prometheus-adapter`

## Accessing the UIs

### Port Forwarding (Development/Testing)

#### Prometheus UI
```bash
kubectl port-forward -n monitoring svc/prometheus-k8s 9090:9090
```
Access at: http://localhost:9090

#### Grafana
```bash
kubectl port-forward -n monitoring svc/grafana 3000:3000
```
Access at: http://localhost:3000
- Default credentials: `admin` / `admin`

#### Alertmanager
```bash
kubectl port-forward -n monitoring svc/alertmanager-main 9093:9093
```
Access at: http://localhost:9093

### Exposing via Load Balancer (Production)

For production access on GKE, you can expose services via LoadBalancer or Ingress. Example for Grafana:

```bash
kubectl patch svc grafana -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'

# Get the external IP (may take a minute)
kubectl get svc grafana -n monitoring
```

**Security Note**: For production, always secure these endpoints with authentication, TLS, and network policies.

## Customization

To customize the monitoring stack (e.g., retention, resources, alerting rules), you should:

1. Review the [kube-prometheus customization documentation](https://github.com/prometheus-operator/kube-prometheus/blob/main/docs/customizing.md)
2. Modify the jsonnet configuration in the original kube-prometheus repo
3. Regenerate manifests: `./build.sh example.jsonnet`
4. Copy updated manifests to this directory

## Monitoring Your Application

To monitor the hello-world application deployed by this repository:

1. Add Prometheus annotations to the app's service:
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "3000"
```

2. Or create a ServiceMonitor resource (recommended):
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: hello-world-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: hello-world
  endpoints:
  - port: http
    interval: 30s
```

## Troubleshooting

### Pods Not Starting
```bash
# Check pod status and events
kubectl describe pod -n monitoring POD_NAME

# Check logs
kubectl logs -n monitoring POD_NAME
```

### CRDs Not Established
```bash
# List CRDs
kubectl get customresourcedefinitions | grep monitoring.coreos.com

# Verify CRD status
kubectl get crd prometheuses.monitoring.coreos.com -o yaml
```

### Prometheus Not Scraping Targets
1. Check ServiceMonitor configuration
2. Verify RBAC permissions
3. Check Prometheus logs: `kubectl logs -n monitoring prometheus-k8s-0 prometheus`

### GKE-Specific Issues

**Insufficient Resources**:
```bash
# Check node resources
kubectl top nodes
kubectl describe nodes
```

**RBAC Permissions**:
```bash
# Grant yourself cluster-admin (if needed)
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole=cluster-admin \
  --user=$(gcloud config get-value account)
```

## Cleanup

To remove the monitoring stack:

```bash
# Remove all monitoring components
kubectl delete -f monitoring/manifests/

# Remove CRDs and namespace
kubectl delete -f monitoring/setup/
```

**Warning**: This will delete all monitoring data and cannot be undone.

## Additional Resources

- [kube-prometheus Documentation](https://github.com/prometheus-operator/kube-prometheus)
- [Prometheus Operator Documentation](https://prometheus-operator.dev/)
- [Grafana Dashboards](https://grafana.com/grafana/dashboards/)
- [GKE Monitoring Best Practices](https://cloud.google.com/stackdriver/docs/solutions/gke)

## Version Compatibility

This setup is tested with:
- kube-prometheus release-0.16
- Kubernetes 1.31-1.34
- GKE Standard and Autopilot clusters

For other versions, consult the [kube-prometheus compatibility matrix](https://github.com/prometheus-operator/kube-prometheus#compatibility).

## License

The manifests in this directory are derived from the kube-prometheus project, which is licensed under Apache License 2.0.
