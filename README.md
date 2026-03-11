# 🔭 Observability Stack Helm Chart

온프레미스 쿠버네티스 환경을 위한 **OpenTelemetry + LGTM** 기반 통합 옵저버빌리티 스택입니다.

## 📋 목차

- [아키텍처 개요](#-아키텍처-개요)
- [구성요소](#-구성요소)
- [데이터 흐름](#-데이터-흐름)
- [빠른 시작](#-빠른-시작)
- [환경별 설치](#-환경별-설치)
- [설정 가이드](#-설정-가이드)
- [Values 파라미터](#-values-파라미터)
- [운영 가이드](#-운영-가이드)

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
│  │    Tempo     │ │    Loki     │ │    Mimir    │                            │
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

---

## 🧩 구성요소

### 1. OpenTelemetry Collector (텔레메트리 파이프라인)

| 항목 | 설명 |
|------|------|
| **역할** | 모든 텔레메트리 데이터(Metrics, Logs, Traces)의 수집, 가공, 라우팅을 담당하는 중앙 파이프라인 |
| **이미지** | `otel/opentelemetry-collector-contrib` |
| **포트** | gRPC: `4317`, HTTP: `4318`, Health: `13133`, zPages: `55679`, Metrics: `8888` |
| **배포 모드** | `Deployment` (Gateway) 또는 `DaemonSet` (Agent) |

#### Receivers (수신기)
| Receiver | 설명 | 기본 활성화 |
|----------|------|:-----------:|
| `otlp` | OTLP 프로토콜 (gRPC/HTTP)로 앱에서 직접 수신 | ✅ |
| `prometheus` | Prometheus 방식의 메트릭 스크래핑 (k8s SD 지원) | ✅ |
| `k8s_events` | Kubernetes 이벤트 수집 | ❌ |
| `hostmetrics` | 노드 호스트 메트릭 (CPU, Memory, Disk 등) | ❌ |
| `kubeletstats` | Kubelet에서 노드/파드/컨테이너 메트릭 수집 | ❌ |

#### Processors (처리기)
| Processor | 설명 |
|-----------|------|
| `batch` | 데이터를 배치로 묶어 전송 효율을 높임 |
| `memory_limiter` | OOM Kill 방지를 위한 메모리 사용량 제한 |
| `resource` | 클러스터 이름, 환경 등 공통 속성 주입 |
| `k8sattributes` | K8s 메타데이터(namespace, pod, node 등) 자동 추가 |
| `filter` | 불필요한 메트릭/로그 필터링 |

#### Exporters (전송기)
| Exporter | 대상 | 프로토콜 |
|----------|------|----------|
| `prometheusremotewrite` | **Mimir** | Prometheus Remote Write |
| `otlphttp/loki` | **Loki** | OTLP HTTP |
| `otlp/tempo` | **Tempo** | OTLP gRPC |

---

### 2. Grafana Mimir (메트릭 장기 저장소)

| 항목 | 설명 |
|------|------|
| **역할** | Prometheus 호환 장기 메트릭 저장소. 수평 확장 가능한 TSDB로 대규모 메트릭 데이터 처리 |
| **이미지** | `grafana/mimir` |
| **포트** | HTTP: `8080`, gRPC: `9095`, Memberlist: `7946` |
| **워크로드** | StatefulSet |

#### 주요 기능
- **Prometheus 호환**: PromQL 완벽 지원, Prometheus Remote Write API
- **수평 확장**: Memberlist 기반 클러스터링으로 동적 스케일링
- **멀티테넌시**: 조직별 메트릭 격리 지원 (선택적)
- **장기 보존**: 블록 기반 스토리지로 효율적인 장기 데이터 보존
- **고가용성**: Replication factor 설정으로 데이터 안정성 보장

#### 내부 컴포넌트
| 컴포넌트 | 역할 |
|----------|------|
| **Distributor** | 수신된 메트릭을 Ingester로 분배 |
| **Ingester** | 메트릭 데이터 인메모리 저장 후 블록 플러시 |
| **Compactor** | 블록 병합 및 다운샘플링으로 스토리지 최적화 |
| **Store Gateway** | 장기 저장소에서 블록 데이터 조회 |
| **Querier** | PromQL 쿼리 실행 엔진 |

---

### 3. Grafana Loki (로그 수집기)

| 항목 | 설명 |
|------|------|
| **역할** | 인덱스 최적화된 로그 수집·저장·검색 시스템. 라벨 기반으로 로그를 구조화 |
| **이미지** | `grafana/loki` |
| **포트** | HTTP: `3100`, gRPC: `9096` |
| **워크로드** | StatefulSet |

#### 주요 기능
- **라벨 기반 인덱싱**: 전문 인덱싱 대신 라벨만 인덱싱하여 스토리지 효율적
- **LogQL**: Prometheus 스타일의 로그 쿼리 언어
- **멀티테넌시**: Org ID 기반 로그 격리
- **보존 정책**: 자동 만료 및 컴팩션으로 스토리지 관리
- **TSDB 인덱스**: v13 스키마의 TSDB 기반 인덱스로 빠른 쿼리

#### 내부 컴포넌트
| 컴포넌트 | 역할 |
|----------|------|
| **Ingester** | 로그 스트림 수신 및 청크 생성 |
| **Querier** | LogQL 쿼리 실행 및 결과 반환 |
| **Compactor** | 인덱스/청크 병합 및 보존 기간 관리 |

---

### 4. Grafana Tempo (분산 트레이싱)

| 항목 | 설명 |
|------|------|
| **역할** | 분산 트레이싱 백엔드. Span 데이터를 수집·저장하고 TraceID로 조회 |
| **이미지** | `grafana/tempo` |
| **포트** | HTTP: `3200`, gRPC: `9095`, OTLP gRPC: `4317`, OTLP HTTP: `4318` |
| **워크로드** | StatefulSet |

#### 주요 기능
- **TraceID 기반 검색**: 트레이스 ID로 빠른 조회
- **Metrics Generator**: Trace에서 자동으로 RED 메트릭(Rate, Errors, Duration) 생성
- **Service Graph**: 마이크로서비스 간 의존성 자동 시각화
- **Trace-to-Logs**: Trace에서 관련 로그로 바로 이동 (Loki 연동)
- **Trace-to-Metrics**: Trace에서 관련 메트릭으로 연결 (Mimir 연동)

#### 내부 컴포넌트
| 컴포넌트 | 역할 |
|----------|------|
| **Distributor** | 수신된 span을 Ingester로 분배 |
| **Ingester** | Span 데이터 WAL 저장 및 블록 생성 |
| **Compactor** | 블록 병합 및 보존 기간 관리 |
| **Querier** | TraceID 기반 트레이스 조회 |
| **Metrics Generator** | Span → RED 메트릭 자동 변환 후 Mimir로 전송 |

---

### 5. Grafana (시각화 대시보드)

| 항목 | 설명 |
|------|------|
| **역할** | 통합 대시보드 및 시각화 플랫폼. 모든 텔레메트리 데이터를 한 화면에서 탐색 |
| **이미지** | `grafana/grafana` |
| **포트** | HTTP: `3000` |
| **워크로드** | Deployment |

#### 주요 기능
- **통합 Explore**: Metrics, Logs, Traces를 하나의 UI에서 탐색
- **자동 데이터소스 연동**: Mimir, Loki, Tempo 자동 구성
- **Unified Alerting**: 메트릭/로그 기반 통합 알림 시스템
- **Service Map**: Tempo Metrics Generator가 생성한 서비스 맵 시각화
- **Correlations**: Trace ↔ Log ↔ Metric 간 상호 연결 탐색

#### 사전 구성 데이터소스
| 데이터소스 | 타입 | 용도 |
|-----------|------|------|
| **Mimir** | Prometheus | 메트릭 쿼리 (PromQL) |
| **Loki** | Loki | 로그 쿼리 (LogQL) |
| **Tempo** | Tempo | 트레이스 조회 |

---

## 🔄 데이터 흐름

### Metrics (메트릭)
```
Application/K8s → [OTLP/Prometheus] → OTel Collector → [Remote Write] → Mimir → Grafana
                                                                            ↑
                                          Tempo (Metrics Generator) ────────┘
```

### Logs (로그)
```
Application/K8s → [OTLP] → OTel Collector → [OTLP HTTP] → Loki → Grafana
```

### Traces (트레이스)
```
Application → [OTLP] → OTel Collector → [OTLP gRPC] → Tempo → Grafana
```

---

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

## 📊 Values 파라미터

### Global

| Parameter | Description | Default |
|-----------|-------------|---------|
| `global.clusterName` | 클러스터 이름 | `"on-prem"` |
| `global.environment` | 환경 이름 | `"production"` |
| `global.storageClass` | 스토리지 클래스 | `""` |
| `global.imagePullSecrets` | 이미지 풀 시크릿 | `[]` |

### 환경별 비교

| 파라미터 | Minimal | Default | Production |
|----------|---------|---------|------------|
| **Grafana Replicas** | 1 | 1 | 2 |
| **Grafana Memory** | 128~256Mi | 256~512Mi | 512Mi~2Gi |
| **Loki Replicas** | 1 | 1 | 3 |
| **Loki Memory** | 128~512Mi | 256Mi~1Gi | 2~8Gi |
| **Loki Retention** | 24h | 168h | 720h (30d) |
| **Loki Storage** | emptyDir | 10Gi | 100Gi |
| **Tempo Replicas** | 1 | 1 | 3 |
| **Tempo Memory** | 128~512Mi | 256Mi~2Gi | 2~8Gi |
| **Tempo Retention** | 24h | 168h | 336h (14d) |
| **Mimir Replicas** | 1 | 1 | 3 |
| **Mimir Memory** | 256Mi~1Gi | 512Mi~2Gi | 4~16Gi |
| **Mimir Retention** | 24h | 168h | 2160h (90d) |
| **Mimir Storage** | emptyDir | 20Gi | 200Gi |
| **OTel Replicas** | 1 | 1 | 3 |
| **OTel Memory** | 128~256Mi | 256~512Mi | 1~2Gi |
| **Persistence** | ❌ | ✅ | ✅ |
| **Pod Anti-Affinity** | ❌ | ❌ | ✅ |

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

# Mimir 상태 확인
kubectl exec -it <mimir-pod> -n observability -- wget -qO- http://localhost:8080/ready

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

# Mimir 링 상태 확인
kubectl port-forward svc/<release>-observability-stack-mimir 8080:8080 -n observability
curl http://localhost:8080/ingester/ring
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
    ├── mimir/
    │   ├── configmap.yaml              # Mimir 설정
    │   ├── statefulset.yaml            # Mimir StatefulSet
    │   └── service.yaml                # Mimir Service (ClusterIP + Headless) + SA
    └── otel-collector/
        ├── configmap.yaml              # OTel Collector 파이프라인 설정
        ├── deployment.yaml             # OTel Collector Deployment/DaemonSet
        ├── service.yaml                # OTel Collector Service + SA
        └── rbac.yaml                   # ClusterRole + ClusterRoleBinding
```

---

## 📜 라이선스

이 Helm 차트는 내부 사용 목적으로 제작되었습니다.

각 구성요소의 라이선스:
- **Grafana**: AGPL-3.0
- **Loki**: AGPL-3.0
- **Tempo**: AGPL-3.0
- **Mimir**: AGPL-3.0
- **OpenTelemetry Collector**: Apache-2.0
