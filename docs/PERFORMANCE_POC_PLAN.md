# 성능 POC 계획

이 문서는 `values-dev.yaml` 기반의 dev 모드와 `values-prod.yaml` 기반의 prod 모드를 실제 Kubernetes 클러스터에 배포했을 때, 어느 정도의 수집 처리량과 안정성을 보이는지 비교 검증하기 위한 POC 계획입니다.

## 목표

- dev 모드와 prod 모드의 수집 성능 차이를 수치로 확인한다.
- 병목이 Collector, Loki, Tempo, VictoriaMetrics, Grafana 중 어디에서 먼저 발생하는지 식별한다.
- 운영 환경에 필요한 최소 노드 수, CPU/메모리, 스토리지 성능 요구사항을 추정한다.
- 장애 상황에서 데이터 손실과 복구 시간을 확인한다.

## 검증 대상

| 구분 | dev 모드 | prod 모드 |
|------|------------|-----------|
| Values 파일 | `values-dev.yaml` | `values-prod.yaml` |
| 목적 | 개발/사전 검증용 상한선 확인 | 운영 수용량 및 안정성 검증 |
| VictoriaMetrics | single mode | cluster mode (`vminsert=2`, `vmselect=2`, `vmstorage=3`) |
| 배포 위치 | 로컬 또는 소규모 테스트 클러스터 | 실제 운영과 유사한 다중 노드 클러스터 |

## 핵심 질문

- 초당 얼마만큼의 traces, metrics, logs를 안정적으로 수집할 수 있는가
- Collector queue backlog 없이 sustained load를 얼마나 버틸 수 있는가
- Grafana 조회 성능은 어느 시점부터 저하되는가
- MySQL exporter 같은 외부 scrape target을 붙였을 때 수집 지연이 커지는가
- 노드 또는 Pod 장애 시 복구 시간과 데이터 유실 범위는 어느 정도인가

## 측정 지표

### 플랫폼 공통

- CPU 사용률: node, pod, container
- 메모리 RSS/working set
- 디스크 사용량과 IO latency
- 네트워크 throughput
- Pod restart 수
- PVC 사용량 증가 속도

### OpenTelemetry Collector

- OTLP ingest rate
- exporter queue size
- dropped metrics/logs/traces
- batch processor flush latency
- remote write / OTLP export error rate

### Loki

- log ingest throughput
- query latency p50/p95/p99
- chunk flush 주기

### Tempo

- trace ingest throughput
- span drop 여부
- trace query latency
- metrics-generator remote write latency

### VictoriaMetrics

- remote write ingest rate
- query latency p50/p95/p99
- active time series 수
- storage growth rate
- prod 모드에서는 `vminsert`, `vmselect`, `vmstorage`별 CPU/메모리 분리 측정

### Grafana

- dashboard load time
- Explore query latency
- concurrent user 수에 따른 응답시간 변화

## 테스트 시나리오

### 1. Baseline

- 무부하 상태에서 30분 이상 안정적으로 유지되는지 확인
- 기본 리소스 사용량, 저장소 증가량, readiness 변화를 기록

### 2. OTLP Ingest Load

- 샘플 앱 또는 `telemetrygen`으로 traces, metrics, logs를 단계적으로 증가
- 예시 단계:
  - low: 100 req/s
  - medium: 500 req/s
  - high: 1,000 req/s
  - burst: 3,000 req/s 이상
- 각 단계는 최소 15~30분 유지

### 3. Prometheus Scrape Load

- `mysqld_exporter` 같은 외부 exporter 1개에서 시작해 5개, 10개, 20개로 확대
- scrape interval은 `15s`, `30s` 두 가지로 비교
- Collector CPU와 VictoriaMetrics ingest latency 변화를 기록

### 4. Query Load

- Grafana 대시보드 자동 새로고침과 Explore 쿼리를 동시에 실행
- PromQL, LogQL, Trace 조회를 병행해 사용자 체감 성능을 측정
- 동시 사용자 수를 `1`, `5`, `10`, `20`으로 증가

### 5. 장애 복원력

- dev 모드:
  - Collector Pod 재시작
  - Loki/Tempo/VictoriaMetrics Pod 재시작
- prod 모드:
  - `vminsert` 1개 중단
  - `vmselect` 1개 중단
  - `vmstorage` 1개 중단
  - worker node 1개 drain 또는 stop
- 복구 시간, 쿼리 영향, 데이터 유실 여부를 기록

## 부하 생성 도구

| 대상 | 권장 도구 | 비고 |
|------|-----------|------|
| OTLP traces/logs/metrics | `telemetrygen`, 샘플 앱 | 가장 간단한 기준 부하 |
| HTTP 애플리케이션 부하 | `k6`, `vegeta` | 실제 API 트래픽과 연계 가능 |
| Prometheus scrape 부하 | `mysqld_exporter`, 테스트 exporter 다중 배치 | 외부 exporter 시나리오 검증 |
| Grafana 조회 부하 | `k6/browser` 또는 API 호출 스크립트 | dashboard/Explore 응답시간 측정 |

## 실행 절차

### dev 모드

1. `values-dev.yaml`로 배포
2. 기본 상태 30분 관찰
3. OTLP ingest 부하 테스트
4. Grafana query 부하 테스트
5. 외부 exporter scrape 테스트
6. 단일 Pod 장애 복구 테스트

### prod 모드

1. `values-prod.yaml`로 배포
2. `vminsert`, `vmselect`, `vmstorage`가 모두 `Ready`인지 확인
3. OTLP ingest 부하 테스트
4. 외부 exporter scrape 테스트
5. Grafana 동시 조회 테스트
6. `vmstorage`/노드 장애 복구 테스트

## 수집할 결과물

- Helm values 파일 버전
- 클러스터 노드 수와 스펙
- 테스트 구간별 ingest rate
- 구간별 CPU/메모리/디스크/네트워크 사용량
- 쿼리 latency p50/p95/p99
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
|------|------------|-----------|------|
| 최대 안정 OTLP ingest rate |  |  |  |
| 최대 안정 scrape target 수 |  |  |  |
| Grafana p95 query latency |  |  |  |
| VictoriaMetrics storage/day |  |  |  |
| 장애 복구 시간 |  |  |  |
| dropped telemetry |  |  |  |

## 해석 가이드

- dev 모드는 기능 검증과 대략적인 상한선을 보는 용도입니다.
- prod 모드는 실제 운영에 필요한 headroom과 장애 내성을 보는 용도입니다.
- 최종 운영 권장치는 평균값이 아니라 sustained load와 장애 테스트 결과를 기준으로 잡는 것이 맞습니다.
