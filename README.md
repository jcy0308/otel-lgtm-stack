# Observability Stack Helm Chart

An on-premises Kubernetes observability stack built with OpenTelemetry and LGTM components.

## What This Chart Deploys

- Grafana (visualization and alerting)
- Loki (logs)
- Tempo (traces)
- Mimir (metrics, monolithic single-binary)
- OpenTelemetry Collector (ingestion and pipeline)
- Optional VictoriaMetrics (disabled by default and used only when Mimir is disabled)

## Architecture Summary

- Applications send telemetry via OTLP to OpenTelemetry Collector.
- Collector processes and exports:
  - Metrics -> Mimir (default)
  - Logs -> Loki
  - Traces -> Tempo
- Grafana connects to all backends for dashboards and exploration.
- Loki, Tempo, and Mimir use external MinIO object storage configured from `global.objectStorage.minio`.

## Prerequisites

- Kubernetes 1.24+
- Helm 3.10+
- At least 8 GB memory available for dev setup
- A working storage class for persistent volumes
- External MinIO endpoint and credentials

## Quick Start

```bash
cd observability-stack

helm install observability . \
  -n observability \
  --create-namespace

helm status observability -n observability
kubectl get pods -n observability
```

## Environment Install

### Dev

```bash
helm install observability . \
  -f values-dev.yaml \
  -n observability \
  --create-namespace
```

### Prod

```bash
helm install observability . \
  -f values-prod.yaml \
  -n observability \
  --create-namespace
```

### Custom Override

```bash
helm install observability . \
  -f values-prod.yaml \
  -f my-overrides.yaml \
  -n observability \
  --create-namespace
```

## Important Configuration

### 1. External MinIO (Global)

All object-storage backends read from one global block:

```yaml
global:
  objectStorage:
    minio:
      endpoint: "minio.example.com:9000"
      accessKey: "CHANGE_ME_MINIO_ACCESS_KEY"
      secretKey: "CHANGE_ME_MINIO_SECRET_KEY"
      insecure: true
      forcePathStyle: true
      buckets:
        loki: "loki"
        tempo: "tempo"
        mimir: "mimir"
```

### 2. Grafana Ingress (nginx)

Set URL and ingress host in each environment file (`values-dev.yaml`, `values-prod.yaml`):

```yaml
grafana:
  rootUrl: "https://grafana.dev.example.com"
  ingress:
    enabled: true
    ingressClassName: nginx
    host: grafana.dev.example.com
    tls: true
    tlsSecret: grafana-tls
```

### 3. External Scrape Targets

Configure external scrape jobs in environment files under:

`otelCollector.config.prometheusReceiver.externalScrapeTargets`

Supported structure:

```yaml
otelCollector:
  config:
    prometheusReceiver:
      externalScrapeTargets:
        mysql:
          enabled: true
          jobName: onprem-mysql
          scrapeInterval: 30s
          metricsPath: /metrics
          targets:
            - endpoint: 10.10.20.15:9104
              labels:
                service_name: mysql-exporter
        additionalJobs:
          - enabled: true
            jobName: external-node-exporter
            scrapeInterval: 30s
            metricsPath: /metrics
            targets:
              - endpoint: 10.10.20.20:9100
                labels:
                  service_name: node-exporter
```

## Operations

### Port Forward

```bash
kubectl port-forward svc/<release>-observability-stack-grafana 3000:3000 -n observability
kubectl port-forward svc/<release>-observability-stack-otel-collector 55679:55679 -n observability
```

### Health Checks

```bash
kubectl get pods -n observability -o wide

# Loki
kubectl exec -it <loki-pod> -n observability -- wget -qO- http://localhost:3100/ready

# Mimir (default metrics backend)
kubectl exec -it <mimir-pod> -n observability -- wget -qO- http://localhost:9009/ready

# Tempo
kubectl exec -it <tempo-pod> -n observability -- wget -qO- http://localhost:3200/ready

# OTel Collector
kubectl exec -it <otel-pod> -n observability -- wget -qO- http://localhost:13133/
```

### Upgrade

```bash
helm upgrade observability . \
  -f values-prod.yaml \
  -n observability
```

### Uninstall

```bash
helm uninstall observability -n observability
kubectl delete pvc -l app.kubernetes.io/instance=observability -n observability
```

## Repository Layout

```text
observability-stack/
├── Chart.yaml
├── .helmignore
├── values.yaml
├── values-dev.yaml
├── values-prod.yaml
├── README.md
└── templates/
    ├── _helpers.tpl
    ├── namespace.yaml
    ├── NOTES.txt
    ├── grafana/
    ├── loki/
    ├── tempo/
    ├── mimir/
    ├── victoriametrics/
    └── otel-collector/
```
