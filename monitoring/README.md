
# Observability — Prometheus + Grafana

The shortly cluster is monitored by the [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
Helm chart, deployed alongside the application in a dedicated `monitoring`
namespace. This brings the full Prometheus / Grafana / Alertmanager
observability stack with pre-built Kubernetes dashboards.

## Install

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace monitoring

helm install kps prometheus-community/kube-prometheus-stack \
  --namespace monitoring
```

## Access

Grafana:
```bash
kubectl port-forward -n monitoring svc/kps-grafana 3000:80
# then browse to http://localhost:3000
```

Username `admin`. Password from the chart-generated Secret:
```bash
kubectl get secret -n monitoring kps-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Prometheus (raw query UI):
```bash
kubectl port-forward -n monitoring svc/kps-kube-prometheus-stack-prometheus 9090:9090
# then browse to http://localhost:9090
```

## What's collected

- **Node metrics** via `node-exporter` (one DaemonSet pod per node) —
  CPU, memory, disk, network, load average
- **Kubernetes object metrics** via `kube-state-metrics` —
  pod status, deployment status, resource requests/limits, etc.
- **Container runtime metrics** via cAdvisor (built into the kubelet) —
  per-container CPU, memory, network

## Useful PromQL queries

```promql
# Are all scrape targets up?
up

# Per-pod CPU usage rate (1m window) for the shortly app
rate(container_cpu_usage_seconds_total{namespace="shortly", pod=~"shortly-.*"}[1m])

# Memory consumed by each shortly pod (bytes)
container_memory_working_set_bytes{namespace="shortly", pod=~"shortly-.*"}
```

## Future enhancements

- Instrument the shortly Flask app with a `/metrics` endpoint
  (Prometheus client library for Python) to expose application-level
  metrics: request count, latency histogram, error rate.
- Configure Alertmanager with real routing (e.g. Slack webhook).
- Author a custom Grafana dashboard specifically for the shortly service.
- Set up ServiceMonitor CRs to scrape the app's `/metrics` once exposed.