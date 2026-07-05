# devops-opentelemetry

Ready-to-deploy OpenTelemetry observability setup for an AKS cluster using official Helm charts.

This repository contains custom values and helper manifests for deploying:

| Capability | Official Helm chart |
|---|---|
| Metrics, Prometheus Operator, Alertmanager, Grafana | `prometheus-community/kube-prometheus-stack` |
| Logs | `grafana/loki-distributed` |
| Traces | `grafana/tempo` |
| OTLP telemetry ingestion | `open-telemetry/opentelemetry-collector` |

The OpenTelemetry Collector receives telemetry from backend applications over OTLP and forwards:

- **Metrics** to Prometheus
- **Logs** to Loki
- **Traces** to Tempo

Grafana is configured with Prometheus, Loki, and Tempo datasources so you can view service metrics, logs, and traces from one UI.

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

## Repository layout

```text
observability/
  rendered.yaml
  kustomize/
  charts/
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
    README.md
```

## Prerequisites

Install these tools locally or in your CI runner:

- Azure CLI
- kubectl
- Helm 3
- Access to the target AKS cluster
- Backend applications already instrumented with OpenTelemetry SDKs or agents

Check the tools:

```bash
az version
kubectl version --client
helm version
```

## 1. Connect to the AKS cluster

Set these variables for your environment:

```bash
export RESOURCE_GROUP="<your-resource-group>"
export AKS_CLUSTER_NAME="<your-aks-cluster-name>"
```

Get AKS credentials:

```bash
az aks get-credentials \
  --resource-group "$RESOURCE_GROUP" \
  --name "$AKS_CLUSTER_NAME" \
  --overwrite-existing
```

Verify cluster access:

```bash
kubectl cluster-info
kubectl get nodes
```

## 2. Add the official Helm repositories

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
```

## 3. Create the observability namespace

```bash
kubectl create namespace observability --dry-run=client -o yaml | kubectl apply -f -
```

## 4. Deploy kube-prometheus-stack

This installs Prometheus, Grafana, Alertmanager, Prometheus Operator, kube-state-metrics, and node exporter.

```bash
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace observability \
  --values observability/charts/kube-prometheus-stack/values.yaml
```

Wait for the stack to become ready:

```bash
kubectl get pods -n observability
kubectl rollout status deployment/kube-prometheus-stack-grafana -n observability
```

## 5. Deploy Loki Distributed

Loki stores and queries logs exported by the OpenTelemetry Collector.

```bash
helm upgrade --install loki-distributed grafana/loki-distributed \
  --namespace observability \
  --values observability/charts/loki-distributed/values.yaml
```

Validate Loki pods and service:

```bash
kubectl get pods -n observability | grep loki
kubectl get svc -n observability | grep loki
```

## 6. Deploy Tempo

Tempo stores and queries distributed traces exported by the OpenTelemetry Collector.

```bash
helm upgrade --install tempo grafana/tempo \
  --namespace observability \
  --values observability/charts/tempo/values.yaml
```

Validate Tempo:

```bash
kubectl get pods -n observability | grep tempo
kubectl get svc -n observability | grep tempo
```

## 7. Deploy OpenTelemetry Collector

The collector exposes OTLP endpoints for backend applications:

| Protocol | Endpoint inside AKS |
|---|---|
| OTLP gRPC | `opentelemetry-collector.observability.svc.cluster.local:4317` |
| OTLP HTTP | `http://opentelemetry-collector.observability.svc.cluster.local:4318` |

Install the collector:

```bash
helm upgrade --install opentelemetry-collector open-telemetry/opentelemetry-collector \
  --namespace observability \
  --values observability/charts/opentelemetry-collector/values.yaml
```

Validate the collector:

```bash
kubectl get pods -n observability | grep opentelemetry-collector
kubectl get svc -n observability | grep opentelemetry-collector
kubectl logs -n observability deploy/opentelemetry-collector --tail=100
```

## 8. Apply Grafana datasources and dashboard

Apply the datasource and dashboard ConfigMaps:

```bash
kubectl apply -f observability/charts/grafana/datasources.yaml
kubectl apply -f observability/charts/grafana/backend-observability-dashboard.yaml
```

Restart Grafana so it reloads the provisioning ConfigMaps:

```bash
kubectl rollout restart deployment kube-prometheus-stack-grafana -n observability
kubectl rollout status deployment/kube-prometheus-stack-grafana -n observability
```

## 9. Access Grafana

Port-forward Grafana:

```bash
kubectl port-forward -n observability svc/kube-prometheus-stack-grafana 3000:3000
```

Open Grafana:

```text
http://localhost:3000
```

Default credentials are defined in `observability/charts/kube-prometheus-stack/values.yaml`.

For production, replace the default credentials with a Kubernetes Secret, Azure Key Vault, or External Secrets.

## 10. Configure backend applications for OpenTelemetry

Your backend applications must export telemetry to the OpenTelemetry Collector service in the `observability` namespace.

### Option A: OTLP gRPC

Use this for SDKs or agents configured with OTLP gRPC:

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

### Option B: OTLP HTTP/protobuf

Use this for SDKs or agents configured with OTLP HTTP:

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

### Option C: Signal-specific OTLP endpoints

Some SDKs work better with signal-specific endpoints:

```yaml
env:
  - name: OTEL_EXPORTER_OTLP_TRACES_ENDPOINT
    value: "http://opentelemetry-collector.observability.svc.cluster.local:4317"
  - name: OTEL_EXPORTER_OTLP_METRICS_ENDPOINT
    value: "http://opentelemetry-collector.observability.svc.cluster.local:4317"
  - name: OTEL_EXPORTER_OTLP_LOGS_ENDPOINT
    value: "http://opentelemetry-collector.observability.svc.cluster.local:4317"
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: "grpc"
  - name: OTEL_SERVICE_NAME
    value: "backend-service-name"
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: "deployment.environment=dev,service.namespace=backend"
```

For OTLP HTTP signal-specific endpoints, use paths like this:

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
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: "deployment.environment=dev,service.namespace=backend"
```

## 11. Example Kubernetes Deployment integration

Add these environment variables to each backend Deployment.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-backend
  namespace: default
spec:
  template:
    spec:
      containers:
        - name: sample-backend
          image: your-registry/sample-backend:latest
          env:
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://opentelemetry-collector.observability.svc.cluster.local:4317"
            - name: OTEL_EXPORTER_OTLP_PROTOCOL
              value: "grpc"
            - name: OTEL_SERVICE_NAME
              value: "sample-backend"
            - name: OTEL_RESOURCE_ATTRIBUTES
              value: "deployment.environment=dev,service.namespace=backend"
```

Apply your updated backend manifest:

```bash
kubectl apply -f <your-backend-deployment.yaml>
kubectl rollout restart deployment/<your-backend-deployment> -n <your-backend-namespace>
kubectl rollout status deployment/<your-backend-deployment> -n <your-backend-namespace>
```

## 12. Language-specific OpenTelemetry notes

### Java

For Java auto-instrumentation, mount or bake the OpenTelemetry Java agent into the image and set:

```yaml
env:
  - name: JAVA_TOOL_OPTIONS
    value: "-javaagent:/otel/opentelemetry-javaagent.jar"
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://opentelemetry-collector.observability.svc.cluster.local:4317"
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: "grpc"
  - name: OTEL_SERVICE_NAME
    value: "java-backend"
```

### Node.js

For Node.js, initialize OpenTelemetry in your application or preload your instrumentation file:

```yaml
env:
  - name: NODE_OPTIONS
    value: "--require ./instrumentation.js"
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://opentelemetry-collector.observability.svc.cluster.local:4318"
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: "http/protobuf"
  - name: OTEL_SERVICE_NAME
    value: "node-backend"
```

### .NET

For .NET applications, configure the OpenTelemetry SDK exporter endpoint:

```yaml
env:
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://opentelemetry-collector.observability.svc.cluster.local:4317"
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: "grpc"
  - name: OTEL_SERVICE_NAME
    value: "dotnet-backend"
```

### Python

For Python applications using OpenTelemetry auto-instrumentation:

```yaml
env:
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://opentelemetry-collector.observability.svc.cluster.local:4318"
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: "http/protobuf"
  - name: OTEL_SERVICE_NAME
    value: "python-backend"
```

Run the app using your instrumentation entrypoint, for example:

```bash
opentelemetry-instrument python app.py
```

## 13. Validate telemetry ingestion

Generate traffic against your backend services, then check collector logs:

```bash
kubectl logs -n observability deploy/opentelemetry-collector --tail=100
```

Check collector metrics in Prometheus:

```promql
otelcol_receiver_accepted_spans
otelcol_receiver_accepted_metric_points
otelcol_receiver_accepted_log_records
```

Check backend metrics in Prometheus:

```promql
http_server_request_duration_seconds_count
http_server_request_duration_seconds_bucket
```

Check logs in Grafana Explore using Loki:

```logql
{service_name=~".+"}
```

Check traces in Grafana Explore using Tempo:

```text
{ resource.service.name != "" }
```

## 14. Troubleshooting

### Collector is not receiving telemetry

Check the backend environment variables:

```bash
kubectl describe pod <backend-pod-name> -n <backend-namespace> | grep OTEL
```

Check that the collector service exists:

```bash
kubectl get svc opentelemetry-collector -n observability
```

From a backend pod, test DNS resolution:

```bash
kubectl exec -n <backend-namespace> <backend-pod-name> -- nslookup opentelemetry-collector.observability.svc.cluster.local
```

### Traces are missing in Grafana

Check collector span counters:

```promql
otelcol_receiver_accepted_spans
```

Check Tempo service and logs:

```bash
kubectl get svc tempo -n observability
kubectl logs -n observability deploy/tempo --tail=100
```

### Logs are missing in Grafana

Check collector log counters:

```promql
otelcol_receiver_accepted_log_records
```

Check Loki service and logs:

```bash
kubectl get svc -n observability | grep loki
kubectl logs -n observability deploy/loki-distributed-gateway --tail=100
```

### Metrics are missing in Prometheus

Check the collector Prometheus exporter endpoint:

```bash
kubectl port-forward -n observability svc/opentelemetry-collector 8889:8889
```

Open:

```text
http://localhost:8889/metrics
```

Also check Prometheus targets:

```bash
kubectl port-forward -n observability svc/kube-prometheus-stack-prometheus 9090:9090
```

Open:

```text
http://localhost:9090/targets
```

## 15. Uninstall

```bash
helm uninstall opentelemetry-collector -n observability
helm uninstall tempo -n observability
helm uninstall loki-distributed -n observability
helm uninstall kube-prometheus-stack -n observability
```

Remove namespace if no longer needed:

```bash
kubectl delete namespace observability
```

## Production recommendations

Before using this in production:

- Replace default Grafana credentials.
- Enable persistent storage for Prometheus, Loki, and Tempo where needed.
- Consider Azure Blob Storage for Loki and Tempo long-term retention.
- Restrict Grafana access using private ingress, Entra ID, OAuth, or IP allowlists.
- Tune retention and resource requests based on telemetry volume.
- Add alerts for error rate, latency, pod restarts, collector dropped spans, and Loki/Tempo ingestion failures.
