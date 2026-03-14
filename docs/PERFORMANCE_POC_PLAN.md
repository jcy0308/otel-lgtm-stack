# 성능 POC 실행 가이드 (3노드 온프렘 Kubernetes)

이 문서는 `values-dev.yaml`과 `values-prod.yaml`를 기준으로, 3노드 온프렘 Kubernetes에서 Observability 스택 성능 POC를 **재현 가능**하게 수행하기 위한 실행 가이드입니다.

핵심 원칙:
- 먼저 `dev`에서 병목 자원(CPU/메모리/디스크 IO/네트워크) 위치를 찾는다.
- 같은 부하 프로파일을 `prod`에 적용해 오버헤드를 정량화한다.
- `VM/Loki/Tempo`는 각각 임계치 테스트를 별도로 수행한다.
- 모든 단계에서 명령어 단위로 결과를 수집한다.

## 범위와 목표

- `dev`와 `prod` 간 최대 안정 처리량 차이 확인
- 자원 병목의 최초 발생 지점 확인
- 장애 상황에서 복구 시간(RTO)과 데이터 손실 유무 확인

## 비교 프레임워크 (중요)

`prod`는 HA/복제 구조라 동일 입력 부하에서도 자원 사용량이 더 큰 것이 정상입니다. 따라서 아래 3트랙으로 분리해 비교합니다.

| 트랙 | 목적 | 비교 방식 |
|---|---|---|
| Track A: 동일 입력 부하 | HA 오버헤드 측정 | dev/prod에 같은 ingest rate 주입 후 자원 사용량 비교 |
| Track B: 최대 처리량 | 실질 수용량 측정 | dev/prod 각각 부하를 단계적으로 올려 임계치 도달점 확인 |
| Track C: 장애 복원력 | 안정성 비교 | 각 모드 `최대 처리량의 60~70%` 부하에서 장애 주입 후 복구시간 비교 |

오버헤드 계산 예시:

```text
CPU Overhead(%) = ((prod_cpu - dev_cpu) / dev_cpu) * 100
Memory Overhead(%) = ((prod_mem - dev_mem) / dev_mem) * 100
```

## 클러스터 전제

- 노드: 3대 (control-plane 1 + worker 2)
- 스토리지: `values`에 정의된 storage class 사용
- prod에서 Loki/Tempo는 MinIO(S3 API) 기반 shared storage 사용
- K8s metrics: `metrics-server` 활성화 필요 (`kubectl top` 사용)

## 측정 항목과 명령어

| 카테고리 | 측정 항목 | 기본 명령어 |
|---|---|---|
| Node 자원 | CPU/Memory 사용량 | `kubectl top nodes` |
| Pod 자원 | 컴포넌트별 CPU/Memory | `kubectl top pods -n <ns>` |
| 스토리지 | PVC 사용 추이/상태 | `kubectl get pvc -n <ns>` |
| 안정성 | Pod 재시작/비정상 상태 | `kubectl get pods -n <ns> -o wide` |
| Collector | drop/export error | `kubectl logs -n <ns> <collector-pod>` |
| 쿼리 성능 | Grafana API 응답, VM query 응답 | `curl` 기반 반복 호출 |
| 장애복원 | 재시작 후 Ready 복귀 시간 | `kubectl rollout status ... --timeout=...` |

### PromQL로 자원 병목 관찰 (권장)

dev(single VM):

```bash
curl -sG "http://127.0.0.1:8428/api/v1/query" \
  --data-urlencode 'query=avg(100 - rate(node_cpu_seconds_total{mode="idle"}[5m]) * 100) by (instance)'
```

prod(cluster VMselect):

```bash
curl -sG "http://127.0.0.1:8481/select/0/prometheus/api/v1/query" \
  --data-urlencode 'query=avg(100 - rate(node_cpu_seconds_total{mode="idle"}[5m]) * 100) by (instance)'
```

메모리/디스크/네트워크 예시:

```bash
# 메모리 사용률
QUERY='(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100'

# 디스크 IO throughput
QUERY='sum(rate(node_disk_read_bytes_total[5m]) + rate(node_disk_written_bytes_total[5m])) by (instance)'

# 네트워크 throughput
QUERY='sum(rate(node_network_receive_bytes_total[5m]) + rate(node_network_transmit_bytes_total[5m])) by (instance)'
```

## 공통 준비

```bash
cd /Users/jcy/scg/otel-lgtm-stack
export KUBECONFIG=~/.kube/config:~/.kube/config-kind

export DEV_RELEASE=observability-dev
export DEV_NS=observability-dev
export PROD_RELEASE=observability
export PROD_NS=observability

mkdir -p /tmp/otel-poc/dev
mkdir -p /tmp/otel-poc/prod
```

```bash
# 권장 부하 단계 (초당 이벤트)
export STAGES="100 300 500 1000 2000 3000 5000"
export STAGE_DURATION=15m
```

```bash
# 사전 점검
kubectl get nodes -o wide
kubectl top nodes
kubectl top pods -A
```

```bash
# 공통 스냅샷 함수
snapshot() {
  local ns="$1"
  local out="$2"
  mkdir -p "$out"
  date > "$out/time.txt"
  kubectl get pods -n "$ns" -o wide > "$out/pods.txt"
  kubectl top pods -n "$ns" > "$out/pods-top.txt"
  kubectl top nodes > "$out/nodes-top.txt"
  kubectl get pvc -n "$ns" > "$out/pvc.txt"
}
```

```bash
# 단일 신호(traces|metrics|logs) 부하 실행 함수
run_signal_stage() {
  local signal="$1"   # traces | metrics | logs
  local rate="$2"
  local duration="$3"
  local release="$4"
  local ns="$5"

  kubectl -n otel-load delete job tg-${signal}-${rate}-${release} --ignore-not-found

  kubectl -n otel-load create job tg-${signal}-${rate}-${release} \
    --image=ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest \
    -- /telemetrygen ${signal} \
    --otlp-endpoint ${release}-observability-stack-otel-collector.${ns}.svc.cluster.local:4317 \
    --otlp-insecure \
    --rate ${rate} \
    --duration ${duration}

  kubectl -n otel-load wait --for=condition=complete job/tg-${signal}-${rate}-${release} --timeout=40m
}
```

## 1) DEV POC (values-dev.yaml)

### 1-1. 배포 및 정상화 확인

```bash
helm upgrade --install "$DEV_RELEASE" . \
  -n "$DEV_NS" \
  --create-namespace \
  -f values-dev.yaml
```

```bash
kubectl rollout status statefulset/${DEV_RELEASE}-observability-stack-loki -n "$DEV_NS" --timeout=300s
kubectl rollout status statefulset/${DEV_RELEASE}-observability-stack-tempo -n "$DEV_NS" --timeout=300s
kubectl rollout status statefulset/${DEV_RELEASE}-observability-stack-vm -n "$DEV_NS" --timeout=300s
kubectl rollout status daemonset/${DEV_RELEASE}-observability-stack-otel-collector -n "$DEV_NS" --timeout=300s
```

### 1-2. Baseline (30분)

```bash
for i in 1 2 3 4 5 6; do
  snapshot "$DEV_NS" "/tmp/otel-poc/dev/baseline-$i"
  sleep 300
done
```

### 1-3. OTLP 부하 테스트 (단계형)

```bash
kubectl create namespace otel-load --dry-run=client -o yaml | kubectl apply -f -
```

```bash
run_stage() {
  local rate="$1"
  local duration="$2"
  kubectl -n otel-load delete job tg-traces-${rate} tg-metrics-${rate} --ignore-not-found

  kubectl -n otel-load create job tg-traces-${rate} \
    --image=ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest \
    -- /telemetrygen traces \
    --otlp-endpoint ${DEV_RELEASE}-observability-stack-otel-collector.${DEV_NS}.svc.cluster.local:4317 \
    --otlp-insecure \
    --rate ${rate} \
    --duration ${duration}

  kubectl -n otel-load create job tg-metrics-${rate} \
    --image=ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest \
    -- /telemetrygen metrics \
    --otlp-endpoint ${DEV_RELEASE}-observability-stack-otel-collector.${DEV_NS}.svc.cluster.local:4317 \
    --otlp-insecure \
    --rate ${rate} \
    --duration ${duration}

  kubectl -n otel-load wait --for=condition=complete job/tg-traces-${rate} --timeout=40m
  kubectl -n otel-load wait --for=condition=complete job/tg-metrics-${rate} --timeout=40m
}
```

```bash
# low, medium, high, burst
run_stage 100 15m
snapshot "$DEV_NS" "/tmp/otel-poc/dev/load-100"

run_stage 500 15m
snapshot "$DEV_NS" "/tmp/otel-poc/dev/load-500"

run_stage 1000 15m
snapshot "$DEV_NS" "/tmp/otel-poc/dev/load-1000"

run_stage 3000 15m
snapshot "$DEV_NS" "/tmp/otel-poc/dev/load-3000"
```

### 1-4. Query 부하 테스트

터미널 A:

```bash
kubectl port-forward -n "$DEV_NS" svc/${DEV_RELEASE}-observability-stack-grafana 3000:3000
```

터미널 B:

```bash
kubectl port-forward -n "$DEV_NS" svc/${DEV_RELEASE}-observability-stack-vm 8428:8428
```

터미널 C:

```bash
export GRAFANA_USER=admin
export GRAFANA_PASS=CHANGE_ME_STRONG_PASSWORD

# Grafana API 반복
for i in $(seq 1 200); do
  curl -s -u ${GRAFANA_USER}:${GRAFANA_PASS} http://127.0.0.1:3000/api/search > /dev/null
done

# VM query 반복
for i in $(seq 1 200); do
  curl -sG "http://127.0.0.1:8428/api/v1/query" --data-urlencode 'query=up' > /dev/null
done

snapshot "$DEV_NS" "/tmp/otel-poc/dev/query-load"
```

### 1-5. 장애복원 테스트

```bash
kubectl rollout restart daemonset/${DEV_RELEASE}-observability-stack-otel-collector -n "$DEV_NS"
kubectl rollout status daemonset/${DEV_RELEASE}-observability-stack-otel-collector -n "$DEV_NS" --timeout=300s

kubectl rollout restart statefulset/${DEV_RELEASE}-observability-stack-loki -n "$DEV_NS"
kubectl rollout status statefulset/${DEV_RELEASE}-observability-stack-loki -n "$DEV_NS" --timeout=300s

kubectl rollout restart statefulset/${DEV_RELEASE}-observability-stack-tempo -n "$DEV_NS"
kubectl rollout status statefulset/${DEV_RELEASE}-observability-stack-tempo -n "$DEV_NS" --timeout=300s

kubectl rollout restart statefulset/${DEV_RELEASE}-observability-stack-vm -n "$DEV_NS"
kubectl rollout status statefulset/${DEV_RELEASE}-observability-stack-vm -n "$DEV_NS" --timeout=300s

snapshot "$DEV_NS" "/tmp/otel-poc/dev/recovery"
```

## 2) PROD POC (values-prod.yaml)

### 2-1. 배포 및 정상화 확인

```bash
helm upgrade --install "$PROD_RELEASE" . \
  -n "$PROD_NS" \
  --create-namespace \
  -f values-prod.yaml
```

```bash
kubectl rollout status statefulset/${PROD_RELEASE}-observability-stack-loki -n "$PROD_NS" --timeout=600s
kubectl rollout status statefulset/${PROD_RELEASE}-observability-stack-tempo -n "$PROD_NS" --timeout=600s
kubectl rollout status deployment/${PROD_RELEASE}-observability-stack-vminsert -n "$PROD_NS" --timeout=600s
kubectl rollout status deployment/${PROD_RELEASE}-observability-stack-vmselect -n "$PROD_NS" --timeout=600s
kubectl rollout status statefulset/${PROD_RELEASE}-observability-stack-vmstorage -n "$PROD_NS" --timeout=600s
kubectl rollout status daemonset/${PROD_RELEASE}-observability-stack-otel-collector -n "$PROD_NS" --timeout=600s
```

### 2-2. Baseline (30분)

```bash
for i in 1 2 3 4 5 6; do
  snapshot "$PROD_NS" "/tmp/otel-poc/prod/baseline-$i"
  sleep 300
done
```

### 2-3. 동일 부하 프로파일 재실행

```bash
run_stage 100 15m
snapshot "$PROD_NS" "/tmp/otel-poc/prod/load-100"

run_stage 500 15m
snapshot "$PROD_NS" "/tmp/otel-poc/prod/load-500"

run_stage 1000 15m
snapshot "$PROD_NS" "/tmp/otel-poc/prod/load-1000"

run_stage 3000 15m
snapshot "$PROD_NS" "/tmp/otel-poc/prod/load-3000"
```

### 2-4. Prod Query endpoint 검증

터미널 A:

```bash
kubectl port-forward -n "$PROD_NS" svc/${PROD_RELEASE}-observability-stack-vm 8481:8481
```

터미널 B:

```bash
curl -sG "http://127.0.0.1:8481/select/0/prometheus/api/v1/query" \
  --data-urlencode 'query=up'

curl -sG "http://127.0.0.1:8481/select/0/prometheus/api/v1/query" \
  --data-urlencode 'query=(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100'
```

### 2-5. 장애복원 테스트 (컴포넌트 + 노드)

```bash
# 컴포넌트 장애
kubectl delete pod -n "$PROD_NS" ${PROD_RELEASE}-observability-stack-loki-0
kubectl delete pod -n "$PROD_NS" ${PROD_RELEASE}-observability-stack-tempo-0
kubectl delete pod -n "$PROD_NS" ${PROD_RELEASE}-observability-stack-vmstorage-0
```

```bash
# 노드 장애 시뮬레이션 (운영 시간 외 권장)
kubectl get nodes -o name
export FAILURE_NODE=<worker-node-name>
kubectl drain ${FAILURE_NODE} --ignore-daemonsets --delete-emptydir-data --force

# 관찰
kubectl get pods -n "$PROD_NS" -o wide
kubectl top nodes
kubectl top pods -n "$PROD_NS"

# 원복
kubectl uncordon ${FAILURE_NODE}
```

```bash
snapshot "$PROD_NS" "/tmp/otel-poc/prod/recovery"
```

## 3) 컴포넌트별 임계치 테스트 (필수)

아래 3개 테스트는 서로 독립적으로 수행합니다.
- VictoriaMetrics 임계치: `metrics` 신호만 주입
- Loki 임계치: `logs` 신호만 주입
- Tempo 임계치: `traces` 신호만 주입

임계치 도달 기준(권장):
- 특정 노드 CPU 또는 메모리 80% 초과가 5분 이상 지속
- 해당 컴포넌트의 drop/error 지표 증가
- 해당 컴포넌트 질의 응답이 급격히 악화
- Pod 재시작 또는 Ready 불안정 발생

### 3-1) VictoriaMetrics 임계치 테스트

목적:
- 메트릭 수집/저장 경로(Collector -> VM)의 최대 안정 처리량 확인

실행 명령:

```bash
# dev (single VM)
kubectl port-forward -n "$DEV_NS" svc/${DEV_RELEASE}-observability-stack-vm 8428:8428

for rate in $STAGES; do
  run_signal_stage metrics "$rate" "$STAGE_DURATION" "$DEV_RELEASE" "$DEV_NS"
  snapshot "$DEV_NS" "/tmp/otel-poc/dev/vm-threshold-${rate}"
  time curl -sG "http://127.0.0.1:8428/api/v1/query" --data-urlencode 'query=up' > /dev/null
done
```

```bash
# prod (cluster VMselect)
kubectl port-forward -n "$PROD_NS" svc/${PROD_RELEASE}-observability-stack-vm 8481:8481

for rate in $STAGES; do
  run_signal_stage metrics "$rate" "$STAGE_DURATION" "$PROD_RELEASE" "$PROD_NS"
  snapshot "$PROD_NS" "/tmp/otel-poc/prod/vm-threshold-${rate}"
  kubectl top pods -n "$PROD_NS" | rg 'vmstorage|vmselect|vminsert|otel-collector'
  time curl -sG "http://127.0.0.1:8481/select/0/prometheus/api/v1/query" --data-urlencode 'query=up' > /dev/null
done
```

### 3-2) Loki 임계치 테스트

목적:
- 로그 ingest/query 경로(Collector -> Loki)의 최대 안정 처리량 확인

실행 명령:

```bash
kubectl port-forward -n "$DEV_NS" svc/${DEV_RELEASE}-observability-stack-loki 3100:3100
kubectl port-forward -n "$PROD_NS" svc/${PROD_RELEASE}-observability-stack-loki 3110:3100
```

```bash
# dev
for rate in $STAGES; do
  run_signal_stage logs "$rate" "$STAGE_DURATION" "$DEV_RELEASE" "$DEV_NS"
  snapshot "$DEV_NS" "/tmp/otel-poc/dev/loki-threshold-${rate}"
  time curl -sG "http://127.0.0.1:3100/loki/api/v1/query" --data-urlencode 'query=count_over_time({} [1m])' > /dev/null
done
```

```bash
# prod
for rate in $STAGES; do
  run_signal_stage logs "$rate" "$STAGE_DURATION" "$PROD_RELEASE" "$PROD_NS"
  snapshot "$PROD_NS" "/tmp/otel-poc/prod/loki-threshold-${rate}"
  kubectl top pods -n "$PROD_NS" | rg 'loki|otel-collector'
  time curl -sG "http://127.0.0.1:3110/loki/api/v1/query" --data-urlencode 'query=count_over_time({} [1m])' > /dev/null
done
```

### 3-3) Tempo 임계치 테스트

목적:
- 트레이스 ingest/store 경로(Collector -> Tempo)의 최대 안정 처리량 확인

실행 명령:

```bash
kubectl port-forward -n "$DEV_NS" svc/${DEV_RELEASE}-observability-stack-tempo 3200:3200
kubectl port-forward -n "$PROD_NS" svc/${PROD_RELEASE}-observability-stack-tempo 3210:3200
```

```bash
# dev
for rate in $STAGES; do
  run_signal_stage traces "$rate" "$STAGE_DURATION" "$DEV_RELEASE" "$DEV_NS"
  snapshot "$DEV_NS" "/tmp/otel-poc/dev/tempo-threshold-${rate}"
  curl -s http://127.0.0.1:3200/metrics | rg 'tempo_distributor_spans_received_total|tempo_distributor_spans_dropped_total' || true
done
```

```bash
# prod
for rate in $STAGES; do
  run_signal_stage traces "$rate" "$STAGE_DURATION" "$PROD_RELEASE" "$PROD_NS"
  snapshot "$PROD_NS" "/tmp/otel-poc/prod/tempo-threshold-${rate}"
  kubectl top pods -n "$PROD_NS" | rg 'tempo|otel-collector'
  curl -s http://127.0.0.1:3210/metrics | rg 'tempo_distributor_spans_received_total|tempo_distributor_spans_dropped_total' || true
done
```

## 결과 정리 템플릿

| 항목 | dev 결과 | prod 결과 | 판정 |
|---|---|---|---|
| 최대 안정 ingest rate |  |  |  |
| VM 최대 안정 rate |  |  |  |
| Loki 최대 안정 rate |  |  |  |
| Tempo 최대 안정 rate |  |  |  |
| 최초 병목 자원 (CPU/Memory/Disk/Network) |  |  |  |
| Collector drop/error 발생 여부 |  |  |  |
| Grafana p95 응답 체감 |  |  |  |
| 장애 복구 시간(RTO) |  |  |  |
| 데이터 유실 여부 |  |  |  |

## 권장 판정 기준

- `kubectl top nodes` 기준 특정 노드 CPU 또는 메모리 80% 이상이 5분 이상 지속되면 병목 경고
- Collector exporter/send_failed 관련 오류가 지속적으로 증가하면 실패
- 장애 테스트 후 핵심 워크로드가 제한 시간 내 Ready 복귀하지 못하면 실패
- prod에서 dev 대비 처리량 증가가 없고 자원 포화만 빨라지면 튜닝 필요
