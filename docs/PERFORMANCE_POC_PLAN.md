# 성능 POC 계획

이 문서는 `values-dev.yaml` 기반의 dev 모드와 `values-prod.yaml` 기반의 prod 모드를 실제 Kubernetes 클러스터에 배포했을 때, 어느 정도의 수집 처리량과 안정성을 보이는지 비교 검증하기 위한 POC 계획입니다. 우선 dev 모드에서 상한선과 병목 자원을 확인한 뒤, 같은 방식으로 prod 모드에서 확장성과 장애 내성을 검증하는 흐름을 전제로 합니다.

## 목표

- dev 모드와 prod 모드의 수집 성능 차이를 수치로 확인한다.
- CPU, 메모리, 디스크 IO, 네트워크 중 어떤 자원이 먼저 병목이 되는지 식별한다.
- 병목 자원이 포화되기 직전의 최대 안정 처리량을 추정한다.
- 운영 환경에 필요한 최소 노드 수, CPU/메모리, 스토리지 성능 요구사항을 추정한다.
- 장애 상황에서 데이터 손실과 복구 시간을 확인한다.

## 검증 대상

| 구분 | dev 모드 | prod 모드 |
|------|----------|-----------|
| Values 파일 | `values-dev.yaml` | `values-prod.yaml` |
| 목적 | 개발/사전 검증용 상한선 확인 | 운영 수용량 및 안정성 검증 |
| Loki / Tempo | single-binary에 가까운 단순 구성 | MinIO shared storage + memberlist 기반 다중 replica 구성 |
| VictoriaMetrics | single mode | cluster mode (`vminsert=2`, `vmselect=2`, `vmstorage=3`) |
| 배포 위치 | 로컬 또는 소규모 테스트 클러스터 | 실제 운영과 유사한 다중 노드 클러스터 |

## 핵심 질문

- 초당 얼마만큼의 traces, metrics, logs를 안정적으로 수집할 수 있는가
- 수집량이 증가할 때 CPU, 메모리, 디스크 IO, 네트워크 중 어떤 자원이 먼저 한계에 도달하는가
- 병목 자원이 포화되기 직전의 최대 안정 처리량은 어느 정도인가
- Collector queue backlog 없이 sustained load를 얼마나 버틸 수 있는가
- Grafana 조회 성능은 어느 시점부터 저하되는가
- 노드 또는 Pod 장애 시 복구 시간과 데이터 유실 범위는 어느 정도인가

## 측정 지표

### 플랫폼 공통

- CPU 사용률: node, pod, container
- 메모리 RSS / working set
- 디스크 사용량과 IO latency
- 네트워크 throughput
- Pod restart 수
- PVC 사용량 증가 속도

### OpenTelemetry Collector

- OTLP ingest rate
- exporter queue size
- dropped metrics / logs / traces
- batch processor flush latency
- remote write / OTLP export error rate

### Loki

- log ingest throughput
- query latency p50 / p95 / p99
- chunk flush 주기

### Tempo

- trace ingest throughput
- span drop 여부
- trace query latency
- metrics-generator remote write latency

### VictoriaMetrics

- remote write ingest rate
- query latency p50 / p95 / p99
- active time series 수
- storage growth rate
- prod 모드에서는 `vminsert`, `vmselect`, `vmstorage`별 CPU / 메모리 분리 측정

### Grafana

- dashboard load time
- Explore query latency
- concurrent user 수에 따른 응답시간 변화

## 사전 준비

```bash
cd /Users/jcy/Desktop/otel-lgtm-stack
export KUBECONFIG=~/.kube/config:~/.kube/config-kind
```

```bash
# metrics-server가 있어야 kubectl top 사용 가능
kubectl top nodes
kubectl top pods -A
```

```bash
# 테스트 결과 저장 디렉터리
mkdir -p /tmp/otel-poc/dev
mkdir -p /tmp/otel-poc/prod
```

## 테스트 시나리오

### 1. Dev Baseline

목적:
- dev 모드의 기본 리소스 사용량과 안정 상태를 확인

실행 명령:

```bash
helm upgrade --install observability-dev . \
  -n observability-dev \
  --create-namespace \
  -f values-dev.yaml
```

```bash
kubectl get pods -n observability-dev
kubectl get svc -n observability-dev
kubectl get pvc -n observability-dev
kubectl top nodes
kubectl top pods -n observability-dev
```

```bash
# 30분 동안 5분 간격으로 스냅샷 수집
for i in 1 2 3 4 5 6; do
  date >> /tmp/otel-poc/dev/baseline-summary.txt
  kubectl top nodes >> /tmp/otel-poc/dev/baseline-node-top.txt
  kubectl top pods -n observability-dev >> /tmp/otel-poc/dev/baseline-pod-top.txt
  kubectl get pods -n observability-dev >> /tmp/otel-poc/dev/baseline-pods.txt
  sleep 300
done
```

### 2. Dev OTLP Ingest Load

목적:
- dev 모드에서 부하를 단계적으로 올리면서 어떤 자원이 먼저 포화되는지 확인

실행 명령:

```bash
kubectl create namespace otel-load --dry-run=client -o yaml | kubectl apply -f -
```

```bash
# low 단계: traces 100/sec, 15분
kubectl -n otel-load run telemetrygen-traces-low \
  --image=ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest \
  --restart=Never \
  -- /telemetrygen traces \
  --otlp-endpoint observability-dev-observability-stack-otel-collector.observability-dev.svc.cluster.local:4317 \
  --otlp-insecure \
  --rate 100 \
  --duration 15m
```

```bash
# low 단계: metrics 100/sec, 15분
kubectl -n otel-load run telemetrygen-metrics-low \
  --image=ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest \
  --restart=Never \
  -- /telemetrygen metrics \
  --otlp-endpoint observability-dev-observability-stack-otel-collector.observability-dev.svc.cluster.local:4317 \
  --otlp-insecure \
  --rate 100 \
  --duration 15m
```

```bash
# 단계별 관측 명령
kubectl top nodes
kubectl top pods -n observability-dev
kubectl logs -n observability-dev -l app.kubernetes.io/name=otel-collector --tail=200
kubectl logs -n observability-dev -l app.kubernetes.io/name=loki --tail=200
kubectl logs -n observability-dev -l app.kubernetes.io/name=tempo --tail=200
```

```bash
# medium / high / burst는 rate만 바꿔 반복
# medium: 500/sec
# high: 1000/sec
# burst: 3000/sec
kubectl -n otel-load delete pod telemetrygen-traces-low telemetrygen-metrics-low --ignore-not-found
```

```bash
# 각 단계 결과 기록
date >> /tmp/otel-poc/dev/ingest-summary.txt
kubectl top nodes >> /tmp/otel-poc/dev/ingest-node-top.txt
kubectl top pods -n observability-dev >> /tmp/otel-poc/dev/ingest-pod-top.txt
```

### 3. Dev Query Load

목적:
- Grafana 조회 부하가 걸릴 때 어떤 자원이 먼저 증가하는지 확인

실행 명령:

```bash
kubectl port-forward -n observability-dev svc/observability-dev-observability-stack-grafana 3000:3000
```

```bash
kubectl port-forward -n observability-dev svc/observability-dev-observability-stack-vm 8428:8428
```

```bash
# Grafana health / datasource 확인
curl -u admin:CHANGE_ME_STRONG_PASSWORD http://127.0.0.1:3000/api/health
curl -u admin:CHANGE_ME_STRONG_PASSWORD http://127.0.0.1:3000/api/datasources
```

```bash
# VictoriaMetrics query 반복
for i in $(seq 1 100); do
  curl -s "http://127.0.0.1:8428/api/v1/query?query=up" > /dev/null
done
```

```bash
# Grafana API 반복 호출
for i in $(seq 1 50); do
  curl -s -u admin:CHANGE_ME_STRONG_PASSWORD http://127.0.0.1:3000/api/search > /dev/null
done
```

```bash
# Query 부하 중 리소스 상태 확인
kubectl top nodes
kubectl top pods -n observability-dev
```

### 4. Dev 장애 복원력

목적:
- 단일 Pod 재시작 시 데이터 수집과 조회가 얼마나 빨리 정상화되는지 확인

실행 명령:

```bash
# Collector 재시작
kubectl rollout restart deployment/observability-dev-observability-stack-otel-collector -n observability-dev
kubectl rollout status deployment/observability-dev-observability-stack-otel-collector -n observability-dev --timeout=180s
```

```bash
# Loki / Tempo / VictoriaMetrics 재시작
kubectl rollout restart statefulset/observability-dev-observability-stack-loki -n observability-dev
kubectl rollout restart statefulset/observability-dev-observability-stack-tempo -n observability-dev
kubectl rollout restart statefulset/observability-dev-observability-stack-vm -n observability-dev
```

```bash
kubectl rollout status statefulset/observability-dev-observability-stack-loki -n observability-dev --timeout=180s
kubectl rollout status statefulset/observability-dev-observability-stack-tempo -n observability-dev --timeout=180s
kubectl rollout status statefulset/observability-dev-observability-stack-vm -n observability-dev --timeout=180s
```

```bash
# 장애 후 상태 수집
kubectl get pods -n observability-dev
kubectl top pods -n observability-dev
kubectl logs -n observability-dev -l app.kubernetes.io/name=otel-collector --tail=200
```

### 5. Prod 검증 확장

목적:
- dev에서 확인한 부하 패턴을 prod 토폴로지에 적용해 확장성과 장애 내성을 검증

실행 명령:

```bash
helm upgrade --install observability . \
  -n observability \
  --create-namespace \
  -f values-prod.yaml
```

```bash
kubectl get pods -n observability
kubectl top pods -n observability
kubectl get pvc -n observability
```

```bash
# prod 핵심 워크로드 확인
kubectl get pods -n observability -l app.kubernetes.io/component=victoria-metrics
kubectl get pods -n observability -l app.kubernetes.io/name=loki
kubectl get pods -n observability -l app.kubernetes.io/name=tempo
```

```bash
# prod 장애 시나리오 예시
kubectl delete pod -n observability -l app.kubernetes.io/name=loki --field-selector=status.phase=Running --wait=false
kubectl delete pod -n observability -l app.kubernetes.io/name=tempo --field-selector=status.phase=Running --wait=false
kubectl delete pod -n observability -l app.kubernetes.io/name=vmstorage --field-selector=status.phase=Running --wait=false
```

```bash
# 장애 후 상태 및 자원 재확인
kubectl get pods -n observability
kubectl top nodes
kubectl top pods -n observability
```

## 부하 생성 도구

| 대상 | 권장 도구 | 비고 |
|------|-----------|------|
| OTLP traces / logs / metrics | `telemetrygen`, 샘플 앱 | 가장 간단한 기준 부하 |
| HTTP 애플리케이션 부하 | `k6`, `vegeta` | 실제 API 트래픽과 연계 가능 |
| Grafana 조회 부하 | `k6/browser` 또는 API 호출 스크립트 | dashboard / Explore 응답시간 측정 |

## 실행 순서

### dev 모드

1. `values-dev.yaml`로 배포
2. Baseline 측정
3. OTLP ingest 부하 테스트
4. Grafana / VM query 부하 테스트
5. 단일 Pod 장애 복구 테스트

### prod 모드

1. `values-prod.yaml`로 배포
2. `Loki`, `Tempo`, `vminsert`, `vmselect`, `vmstorage`가 모두 `Ready`인지 확인
3. dev에서 사용한 부하 패턴을 동일하게 재실행
4. `Loki`, `Tempo`, `vmstorage`, 노드 장애 복구 테스트

## 수집할 결과물

- Helm values 파일 버전
- 클러스터 노드 수와 스펙
- 테스트 구간별 ingest rate
- 구간별 CPU / 메모리 / 디스크 / 네트워크 사용량
- 쿼리 latency p50 / p95 / p99
- 에러율과 dropped telemetry 수
- 장애 유도 시 복구 시간
- 최종 권장 운영 한계치

## 판정 기준 예시

- 1시간 sustained load 동안 dropped telemetry가 0 또는 허용 범위 이내
- p95 query latency가 목표 SLA 이내
- Pod restart 없이 안정적으로 유지
- prod 모드에서 단일 Pod 또는 단일 노드 장애 후 서비스 지속 가능
- 저장소 증가율이 운영 보존 정책과 디스크 용량 계획에 부합

## 권장 산출물 템플릿

| 항목 | dev 결과 | prod 결과 | 비고 |
|------|----------|-----------|------|
| 최대 안정 OTLP ingest rate |  |  |  |
| Grafana p95 query latency |  |  |  |
| VictoriaMetrics storage/day |  |  |  |
| 장애 복구 시간 |  |  |  |
| dropped telemetry |  |  |  |
| 최초 병목 자원 |  |  | CPU / Memory / Disk / Network |

## 해석 가이드

- dev 모드는 기능 검증과 대략적인 상한선을 보는 용도입니다.
- prod 모드는 실제 운영에 필요한 headroom과 장애 내성을 보는 용도입니다.
- 최종 운영 권장치는 평균값이 아니라 sustained load와 장애 테스트 결과를 기준으로 잡는 것이 맞습니다.
