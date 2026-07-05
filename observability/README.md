# Observability Charts

This directory vendors chart folders for the observability stack and keeps repository-specific configuration in `custom-values.yaml` files.

Do not edit the built-in `values.yaml` files for environment changes. Use `custom-values.yaml` for overrides.

## Layout

```text
observability/charts/
  kube-prometheus-stack/
    Chart.yaml
    values.yaml
    custom-values.yaml
  loki-distributed/
    Chart.yaml
    values.yaml
    custom-values.yaml
  tempo/
    Chart.yaml
    values.yaml
    custom-values.yaml
  opentelemetry-collector/
    Chart.yaml
    values.yaml
    custom-values.yaml
  grafana/
    datasources.yaml
    backend-observability-dashboard.yaml
```

## Refresh upstream built-in values

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

helm show values prometheus-community/kube-prometheus-stack > observability/charts/kube-prometheus-stack/values.yaml
helm show values grafana/loki-distributed > observability/charts/loki-distributed/values.yaml
helm show values grafana/tempo > observability/charts/tempo/values.yaml
helm show values open-telemetry/opentelemetry-collector > observability/charts/opentelemetry-collector/values.yaml
```

## Deploy with custom values

```bash
kubectl create namespace observability --dry-run=client -o yaml | kubectl apply -f -

helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace observability \
  --values observability/charts/kube-prometheus-stack/custom-values.yaml

helm upgrade --install loki-distributed grafana/loki-distributed \
  --namespace observability \
  --values observability/charts/loki-distributed/custom-values.yaml

helm upgrade --install tempo grafana/tempo \
  --namespace observability \
  --values observability/charts/tempo/custom-values.yaml

helm upgrade --install opentelemetry-collector open-telemetry/opentelemetry-collector \
  --namespace observability \
  --values observability/charts/opentelemetry-collector/custom-values.yaml

kubectl apply -f observability/charts/grafana/datasources.yaml
kubectl apply -f observability/charts/grafana/backend-observability-dashboard.yaml
kubectl rollout restart deployment kube-prometheus-stack-grafana -n observability
```
