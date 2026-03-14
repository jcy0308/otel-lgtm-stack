# 🔭 Observability Stack Helm Chart

온프레미스 쿠버네티스 환경을 위한 **OpenTelemetry + LGTM** 기반 통합 옵저버빌리티 스택입니다.

## 📋 목차

- [아키텍처 개요](#-아키텍처-개요)
- [빠른 시작](#-빠른-시작)
- [환경별 설치](#-환경별-설치)
- [설정 가이드](#-설정-가이드)
- [운영 가이드](#-운영-가이드)
- [추가 문서](#-추가-문서)

---

## 🏗 아키텍처 개요

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        On-Premise Kubernetes Cluster                            │
│                                                                                 │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐     │
│  │  Application  │   │  Application  │   │  Application  │   │  K8s Infra   │     │
│  │  (SDK/Auto-   │   │  (SDK/Auto-   │   │  (SDK/Auto-   │   │  Components  │     │
│  │  Instrument)  │   │  Instrument)  │   │  Instrument)  │   │              │     │
│  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘   └──────┬───────┘     │
│         │ OTLP              │ OTLP              │ OTLP             │ Prometheus   │
│         └───────────────────┴───────────────────┴─────────────────┘              │
│                                      │                                           │
│                                      ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐     │
│  │                    OpenTelemetry Collector                               │     │
│  │  ┌─────────────┐   ┌──────────────┐   ┌─────────────────────────────┐  │     │
│  │  │  Receivers   │──▶│  Processors  │──▶│       Exporters             │  │     │
│  │  │             │   │              │   │                             │  │     │
│  │  │ • OTLP      │   │ • Batch      │   │ • PrometheusRemoteWrite ──────────┐  │
│  │  │ • Prometheus │   │ • Memory     │   │ • OTLP/HTTP (Loki)    ────────┐  │  │
│  │  │ • k8s_events│   │   Limiter    │   │ • OTLP (Tempo)        ─────┐  │  │  │
│  │  │ • hostmetric│   │ • Resource   │   │                             │  │  │  │
│  │  │ • kubelet   │   │ • k8sattrib  │   └─────────────────────────────┘  │  │  │
│  │  └─────────────┘   │ • Filter     │                                    │  │  │
│  │                     └──────────────┘                                    │  │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │  │
│                                                                               │  │
│         ┌─────────────────────────────────────────────────────────────────┘  │  │
│         │              ┌──────────────────────────────────────────────────┘  │  │
│         │              │              ┌───────────────────────────────────┘  │
│         ▼              ▼              ▼                                       │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                            │
│  │    Tempo     │ │    Loki     │ │ VictoriaMetrics │                        │
│  │  (Traces)   │ │   (Logs)    │ │  (Metrics)  │                            │
│  │             │ │             │ │             │                            │
│  │ • Distribu- │ │ • Ingester  │ │ • Distribu- │                            │
│  │   tor       │ │ • Querier   │ │   tor       │                            │
│  │ • Ingester  │ │ • Compactor │ │ • Ingester  │                            │
│  │ • Compactor │ │ • Storage   │ │ • Compactor │                            │
│  │ • Querier   │ │   (Local)   │ │ • Store GW  │                            │
│  │ • Metrics   │ │             │ │ • Storage   │                            │
│  │   Generator │ │             │ │   (Local)   │                            │
│  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘                            │
│         │               │               │                                    │
│         └───────────────┼───────────────┘                                    │
│                         │                                                    │
│                         ▼                                                    │
│              ┌─────────────────────┐                                         │
│              │      Grafana        │                                         │
│              │   (Visualization)   │                                         │
│              │                     │                                         │
│              │  • Dashboards       │                                         │
│              │  • Alerting         │                                         │
│              │  • Explore          │                                         │
│              │  • Service Map      │                                         │
│              └──────────┬──────────┘                                         │
│                         │                                                    │
└─────────────────────────┼────────────────────────────────────────────────────┘
                          │
                    ┌─────▼─────┐
                    │  Browser  │
                    │  (Users)  │
                    └───────────┘
```

## 🚀 빠른 시작

### 사전 요구사항

- Kubernetes 1.24+
- Helm 3.10+
- 최소 8GB 메모리 가용 (minimal), 96GB (prod)
- PersistentVolume 프로비저너 (prod)

### 기본 설치

```bash
# 차트 디렉토리로 이동
cd observability-stack

# 기본 설치 (default values)
helm install observability . -n observability --create-namespace

# 설치 확인
helm status observability -n observability
kubectl get pods -n observability
```

---

## 🌍 환경별 설치

### 개발/테스트 환경 (Minimal)

최소 리소스로 빠르게 시작. 퍼시스턴스 없음, 1일 보존.

```bash
helm install observability . \
  -f values-minimal.yaml \
  -n observability \
  --create-namespace
```

| 항목 | 설정 |
|------|------|
| 레플리카 | 모두 1개 |
| 퍼시스턴스 | 비활성화 (emptyDir) |
| 보존 기간 | 24시간 |
| 총 리소스 | ~1.5 CPU, ~1.5GB RAM |
| OTel Receivers | OTLP만 활성화 |

### 프로덕션 환경 (Production)

고가용성, 충분한 리소스, 장기 보존.

```bash
helm install observability . \
  -f values-prod.yaml \
  -n observability \
  --create-namespace
```

| 항목 | 설정 |
|------|------|
| 레플리카 | 2~3개 (HA) |
| 퍼시스턴스 | 활성화 (100~200GB) |
| 보존 기간 | Metrics 90일, Logs 30일, Traces 14일 |
| 총 리소스 | ~16 CPU, ~40GB RAM (requests) |
| OTel Receivers | 모든 receiver 활성화 |
| 스케줄링 | 전용 노드 + Pod Anti-Affinity |
| Ingress | TLS 활성화 |

### 커스텀 설치

여러 values 파일 조합:

```bash
helm install observability . \
  -f values-prod.yaml \
  -f my-overrides.yaml \
  -n observability \
  --create-namespace
```

---

## ⚙ 설정 가이드

### 온프렘 스토리지 클래스 설정

```yaml
global:
  storageClass: "local-path"  # 또는 "nfs-client", "rook-ceph-block" 등
```

### Grafana 인그레스 설정

```yaml
grafana:
  ingress:
    enabled: true
    ingressClassName: nginx
    host: grafana.mycompany.com
    tls: true
    tlsSecret: grafana-tls-secret
```

### OpenTelemetry Collector를 DaemonSet으로 배포

노드별 호스트 메트릭 수집이 필요한 경우:

```yaml
otelCollector:
  kind: DaemonSet
  hostNetwork: true
  config:
    hostMetricsReceiver:
      enabled: true
    kubeletStatsReceiver:
      enabled: true
```

### 애플리케이션에서 OTel Collector로 전송

```yaml
# 애플리케이션 환경변수 설정
env:
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://<release-name>-observability-stack-otel-collector:4317"
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: "grpc"
  - name: OTEL_SERVICE_NAME
    value: "my-service"
```

### 메트릭 필터링

불필요한 메트릭 제거로 스토리지 절약:

```yaml
otelCollector:
  config:
    filterProcessor:
      enabled: true
      metricFilters:
        - 'name == "unwanted_metric_name"'
        - 'resource.attributes["k8s.namespace.name"] == "kube-system"'
```

---

## 🔧 운영 가이드

### 포트 포워딩 (로컬 접속)

```bash
# Grafana
kubectl port-forward svc/<release>-observability-stack-grafana 3000:3000 -n observability

# OTel Collector (디버깅)
kubectl port-forward svc/<release>-observability-stack-otel-collector 55679:55679 -n observability
# zPages: http://localhost:55679/debug/tracez
```

### 상태 확인

```bash
# 모든 Pod 상태 확인
kubectl get pods -n observability -o wide

# Loki 상태 확인
kubectl exec -it <loki-pod> -n observability -- wget -qO- http://localhost:3100/ready

# VictoriaMetrics 상태 확인
# single mode
kubectl exec -it <vm-pod> -n observability -- wget -qO- http://localhost:8428/health

# cluster mode
kubectl exec -it <vmselect-pod> -n observability -- wget -qO- http://localhost:8481/health
kubectl exec -it <vminsert-pod> -n observability -- wget -qO- http://localhost:8480/health
kubectl exec -it <vmstorage-pod> -n observability -- wget -qO- http://localhost:8482/health

# Tempo 상태 확인
kubectl exec -it <tempo-pod> -n observability -- wget -qO- http://localhost:3200/ready

# OTel Collector 상태 확인
kubectl exec -it <otel-pod> -n observability -- wget -qO- http://localhost:13133/
```

### 업그레이드

```bash
helm upgrade observability . \
  -f values-prod.yaml \
  -n observability
```

### 삭제

```bash
helm uninstall observability -n observability

# PVC도 삭제하려면
kubectl delete pvc -l app.kubernetes.io/instance=observability -n observability
```

### 트러블슈팅

```bash
# OTel Collector 로그 확인
kubectl logs -l app.kubernetes.io/name=otel-collector -n observability --tail=100

# OTel Collector 파이프라인 메트릭
kubectl port-forward svc/<release>-observability-stack-otel-collector 8888:8888 -n observability
curl http://localhost:8888/metrics | grep otelcol_exporter

# Loki 로그 확인
kubectl logs -l app.kubernetes.io/name=loki -n observability --tail=100

# VictoriaMetrics 메트릭 확인
# single mode
kubectl port-forward svc/<release>-observability-stack-vm 8428:8428 -n observability
curl http://localhost:8428/metrics

# cluster mode query endpoint
kubectl port-forward svc/<release>-observability-stack-vm 8481:8481 -n observability
curl "http://localhost:8481/select/0/prometheus/api/v1/query?query=up"
```

---

## 📁 차트 구조

```
observability-stack/
├── Chart.yaml                          # 차트 메타데이터
├── .helmignore                         # Helm 빌드 시 무시 패턴
├── values.yaml                         # 기본 Values
├── values-minimal.yaml                 # 개발/테스트용 최소 Values
├── values-prod.yaml                    # 프로덕션용 Values
├── README.md                           # 문서
└── templates/
    ├── _helpers.tpl                    # 템플릿 헬퍼 함수
    ├── namespace.yaml                  # 네임스페이스
    ├── NOTES.txt                       # 설치 후 안내 메시지
    ├── grafana/
    │   ├── configmap.yaml              # Grafana 설정 + 데이터소스
    │   ├── deployment.yaml             # Grafana Deployment
    │   ├── service.yaml                # Grafana Service + SA
    │   ├── ingress.yaml                # Grafana Ingress
    │   └── pvc.yaml                    # Grafana PVC
    ├── loki/
    │   ├── configmap.yaml              # Loki 설정
    │   ├── statefulset.yaml            # Loki StatefulSet
    │   └── service.yaml                # Loki Service (ClusterIP + Headless) + SA
    ├── tempo/
    │   ├── configmap.yaml              # Tempo 설정
    │   ├── statefulset.yaml            # Tempo StatefulSet
    │   └── service.yaml                # Tempo Service (ClusterIP + Headless) + SA
    ├── victoriametrics/
    │   ├── configmap.yaml              # VictoriaMetrics placeholder 설정
    │   ├── statefulset.yaml            # VictoriaMetrics single/cluster workload 템플릿
    │   └── service.yaml                # VictoriaMetrics single/cluster Service + SA
    └── otel-collector/
        ├── configmap.yaml              # OTel Collector 파이프라인 설정
        ├── deployment.yaml             # OTel Collector Deployment/DaemonSet
        ├── service.yaml                # OTel Collector Service + SA
        └── rbac.yaml                   # ClusterRole + ClusterRoleBinding
```

---

## 📚 추가 문서

- [성능 POC 계획](./docs/PERFORMANCE_POC_PLAN.md)
- [SDK 연동 가이드](./docs/SDK_INTEGRATION.md)
- [자동 계측 가이드](./docs/AUTO_INSTRUMENTATION.md)
- [커스텀 메트릭 가이드](./docs/CUSTOM_METRICS.md)
