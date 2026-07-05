# DevOps OpenTelemetry Observability Charts

This folder contains custom values for official Helm charts, based on the existing manifests in this repository.

## Official Helm repositories

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
```

## Folder structure

```text
observability/charts/
  kube-prometheus-stack/
    values.yaml
  loki-distributed/
    values.yaml
  tempo/
    values.yaml
  opentelemetry-collector/
    values.yaml
  grafana/
    datasources.yaml
    backend-observability-dashboard.yaml
```

## Deploy

Create the namespace:

```bash
kubectl create namespace observability --dry-run=client -o yaml | kubectl apply -f -
```

Install kube-prometheus-stack:

```bash
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace observability \
  --values observability/charts/kube-prometheus-stack/values.yaml
```

Install Loki Distributed:

```bash
helm upgrade --install loki-distributed grafana/loki-distributed \
  --namespace observability \
  --values observability/charts/loki-distributed/values.yaml
```

Install Tempo:

```bash
helm upgrade --install tempo grafana/tempo \
  --namespace observability \
  --values observability/charts/tempo/values.yaml
```

Install OpenTelemetry Collector:

```bash
helm upgrade --install opentelemetry-collector open-telemetry/opentelemetry-collector \
  --namespace observability \
  --values observability/charts/opentelemetry-collector/values.yaml
```

Apply Grafana provisioning manifests:

```bash
kubectl apply -f observability/charts/grafana/datasources.yaml
kubectl apply -f observability/charts/grafana/backend-observability-dashboard.yaml
kubectl rollout restart deployment kube-prometheus-stack-grafana -n observability
```

## Backend application OpenTelemetry endpoint

For OTLP gRPC:

```yaml
env:
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://opentelemetry-collector.observability.svc.cluster.local:4317"
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: "grpc"
  - name: OTEL_SERVICE_NAME
    value: "your-service-name"
```

For OTLP HTTP:

```yaml
env:
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://opentelemetry-collector.observability.svc.cluster.local:4318"
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: "http/protobuf"
  - name: OTEL_SERVICE_NAME
    value: "your-service-name"
```

## Validate

```bash
kubectl get pods -n observability
kubectl get svc -n observability
kubectl get servicemonitor -n observability
```

Open Grafana locally:

```bash
kubectl port-forward -n observability svc/kube-prometheus-stack-grafana 3000:3000
```

Then open:

```text
http://localhost:3000
```

## Notes

- The current values keep local filesystem-style storage behavior close to the original manifests.
- For production AKS, configure persistent storage or Azure Blob Storage for Loki and Tempo before increasing retention.
- Replace default Grafana admin credentials before production use.
