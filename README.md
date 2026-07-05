# devops-opentelemetry

Ready-to-deploy OpenTelemetry observability setup for an AKS cluster using official Helm charts.

This repository stores chart folders under `observability/charts`. Each chart has a built-in `values.yaml` file and a separate `custom-values.yaml` file for AKS/backend-specific overrides.

Do not alter the built-in `values.yaml` files for environment changes. Put custom configuration in `custom-values.yaml`.

## Components

| Capability | Official Helm chart | Custom override file |
|---|---|---|
| Metrics, Prometheus Operator, Alertmanager, Grafana | `prometheus-community/kube-prometheus-stack` | `observability/charts/kube-prometheus-stack/custom-values.yaml` |
| Logs | `grafana/loki-distributed` | `observability/charts/loki-distributed/custom-values.yaml` |
| Traces | `grafana/tempo` | `observability/charts/tempo/custom-values.yaml` |
| OTLP telemetry ingestion | `open-telemetry/opentelemetry-collector` | `observability/charts/opentelemetry-collector/custom-values.yaml` |

## Architecture

```text
Backend Applications
  OpenTelemetry SDK / Agent
        |
        | OTLP gRPC 4317 or OTLP HTTP 4318
        v
OpenTelemetry Collector
        |
        | metrics -> Prometheus
        | logs    -> Loki Distributed
        | traces  -> Tempo
        v
Grafana
```

## Prerequisites

- Azure CLI
- kubectl
- Helm 3
- Access to the target AKS cluster
- Backend applications instrumented with OpenTelemetry SDKs or agents

## Connect to AKS

```bash
export RESOURCE_GROUP="<your-resource-group>"
export AKS_CLUSTER_NAME="<your-aks-cluster-name>"

az aks get-credentials \
  --resource-group "$RESOURCE_GROUP" \
  --name "$AKS_CLUSTER_NAME" \
  --overwrite-existing

kubectl get nodes
```

## Add official Helm repositories

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
```

## Create namespace

```bash
kubectl create namespace observability --dry-run=client -o yaml | kubectl apply -f -
```

## Deploy charts with custom-values.yaml

Deploy kube-prometheus-stack:

```bash
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace observability \
  --values observability/charts/kube-prometheus-stack/custom-values.yaml
```

Deploy Loki Distributed:

```bash
helm upgrade --install loki-distributed grafana/loki-distributed \
  --namespace observability \
  --values observability/charts/loki-distributed/custom-values.yaml
```

Deploy Tempo:

```bash
helm upgrade --install tempo grafana/tempo \
  --namespace observability \
  --values observability/charts/tempo/custom-values.yaml
```

Deploy OpenTelemetry Collector:

```bash
helm upgrade --install opentelemetry-collector open-telemetry/opentelemetry-collector \
  --namespace observability \
  --values observability/charts/opentelemetry-collector/custom-values.yaml
```

Apply Grafana datasources and dashboard:

```bash
kubectl apply -f observability/charts/grafana/datasources.yaml
kubectl apply -f observability/charts/grafana/backend-observability-dashboard.yaml
kubectl rollout restart deployment kube-prometheus-stack-grafana -n observability
```

## Backend application integration

Configure backend workloads to export OTLP telemetry to the collector service.

### OTLP gRPC

```yaml
env:
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://opentelemetry-collector.observability.svc.cluster.local:4317"
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: "grpc"
  - name: OTEL_SERVICE_NAME
    value: "backend-service-name"
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: "deployment.environment=dev,service.namespace=backend"
```

### OTLP HTTP/protobuf

```yaml
env:
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://opentelemetry-collector.observability.svc.cluster.local:4318"
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: "http/protobuf"
  - name: OTEL_SERVICE_NAME
    value: "backend-service-name"
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: "deployment.environment=dev,service.namespace=backend"
```

### Signal-specific HTTP endpoints

```yaml
env:
  - name: OTEL_EXPORTER_OTLP_TRACES_ENDPOINT
    value: "http://opentelemetry-collector.observability.svc.cluster.local:4318/v1/traces"
  - name: OTEL_EXPORTER_OTLP_METRICS_ENDPOINT
    value: "http://opentelemetry-collector.observability.svc.cluster.local:4318/v1/metrics"
  - name: OTEL_EXPORTER_OTLP_LOGS_ENDPOINT
    value: "http://opentelemetry-collector.observability.svc.cluster.local:4318/v1/logs"
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: "http/protobuf"
  - name: OTEL_SERVICE_NAME
    value: "backend-service-name"
```

## Validate telemetry

Check workloads:

```bash
kubectl get pods -n observability
kubectl get svc -n observability
```

Check collector logs:

```bash
kubectl logs -n observability deploy/opentelemetry-collector --tail=100
```

Check collector metrics in Prometheus:

```promql
otelcol_receiver_accepted_spans
otelcol_receiver_accepted_metric_points
otelcol_receiver_accepted_log_records
```

Open Grafana:

```bash
kubectl port-forward -n observability svc/kube-prometheus-stack-grafana 3000:3000
```

Then browse:

```text
http://localhost:3000
```

## Refresh full upstream values.yaml files

The `custom-values.yaml` files are the source of your environment-specific changes. To refresh the chart built-in values from upstream:

```bash
helm show values prometheus-community/kube-prometheus-stack > observability/charts/kube-prometheus-stack/values.yaml
helm show values grafana/loki-distributed > observability/charts/loki-distributed/values.yaml
helm show values grafana/tempo > observability/charts/tempo/values.yaml
helm show values open-telemetry/opentelemetry-collector > observability/charts/opentelemetry-collector/values.yaml
```

## Uninstall

```bash
helm uninstall opentelemetry-collector -n observability
helm uninstall tempo -n observability
helm uninstall loki-distributed -n observability
helm uninstall kube-prometheus-stack -n observability
kubectl delete namespace observability
```
