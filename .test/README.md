# OTel Collector Kanary Metrics - Local Test Environment

Test the full flow: Tekton task pushes OTLP metrics to the OTel collector, which exposes them as Prometheus metrics on port 8889. A ServiceMonitor tells Prometheus to scrape the collector, and Prometheus stores the metrics in its TSDB.

## Prerequisites

- [minikube](https://minikube.sigs.k8s.io/docs/start/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [helm](https://helm.sh/docs/intro/install/)

## Setup

### 1. Start minikube

```bash
minikube start --memory=4096 --cpus=2
```

### 2. Create namespaces

```bash
kubectl apply -f 00-namespaces.yaml
```

### 3. Install Tekton Pipelines

```bash
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
kubectl wait --for=condition=ready pod -l app.kubernetes.io/part-of=tekton-pipelines -n tekton-pipelines --timeout=120s
```

### 4. Install the OTel Collector via Helm

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
helm install otel-collector open-telemetry/opentelemetry-collector \
  -n konflux-otel \
  -f 01-otel-collector-values.yaml
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=opentelemetry-collector -n konflux-otel --timeout=120s
```

### 5. Install Prometheus (kube-prometheus-stack)

This installs the Prometheus Operator and a Prometheus instance that watches ServiceMonitors across all namespaces.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  -f 01b-kube-prometheus-stack-values.yaml
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=prometheus -n monitoring --timeout=180s
```

### 6. Apply the ServiceMonitor

```bash
kubectl apply -f 02-servicemonitor.yaml
```

### 7. Run the Tekton pipeline

```bash
kubectl apply -f 03-tekton-pipeline.yaml
kubectl create -f 04-tekton-pipelinerun.yaml
```

Watch the pipeline run:

```bash
kubectl get pipelinerun -n appstudio-kanary -w
```

Wait for it to complete, then check the task logs:

```bash
kubectl logs -n appstudio-kanary -l tekton.dev/pipelineTask=push-kanary-metrics --tail=50
```

### 8. Verify metrics on the OTel collector

Port-forward to the Prometheus exporter:

```bash
kubectl port-forward -n konflux-otel svc/otel-collector 8889:8889
```

In another terminal, check for kanary metrics:

```bash
curl -s http://localhost:8889/metrics | grep kanary
```

You should see metrics like:

```
kanary_up_poc{...} 1
kanary_error_poc{...} 0
```

### 9. Verify metrics in Prometheus

Port-forward to the Prometheus UI:

```bash
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
```

Open http://localhost:9090 in your browser and query:

```promql
kanary_up_poc
```

You should see the kanary metrics that Prometheus scraped from the OTel collector via the ServiceMonitor.

You can also verify that Prometheus discovered the ServiceMonitor target under **Status > Targets** -- look for a target pointing to `otel-collector.konflux-otel:8889`.

### 10. (Optional) Verify collector internal telemetry

Port-forward to the internal telemetry port:

```bash
kubectl port-forward -n konflux-otel svc/otel-collector 8888:8888
```

Check that the collector received metric data points:

```bash
curl -s http://localhost:8888/metrics | grep otelcol_receiver_accepted_metric_points
```

## Cleanup

```bash
helm uninstall prometheus -n monitoring
helm uninstall otel-collector -n konflux-otel
kubectl delete -f 04-tekton-pipelinerun.yaml --ignore-not-found
kubectl delete -f 03-tekton-pipeline.yaml --ignore-not-found
kubectl delete -f 02-servicemonitor.yaml --ignore-not-found
kubectl delete -f 00-namespaces.yaml
kubectl delete namespace monitoring --ignore-not-found
minikube stop
```
