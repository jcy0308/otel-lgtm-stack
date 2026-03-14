# 📡 애플리케이션 SDK 연동 가이드

이 문서는 애플리케이션 Pod에서 **OpenTelemetry SDK**를 사용하여 Observability Stack의 OTel Collector로 텔레메트리 데이터(Traces, Metrics, Logs)를 전송하는 방법을 설명합니다.

---

## 목차

- [개요](#개요)
- [Collector 엔드포인트](#collector-엔드포인트)
- [환경변수 설정](#환경변수-설정)
- [언어별 SDK 연동](#언어별-sdk-연동)
  - [Java (Spring Boot)](#java-spring-boot)
  - [Python (FastAPI / Flask)](#python-fastapi--flask)
  - [Node.js (Express)](#nodejs-express)
  - [Go](#go)
- [Kubernetes Deployment 예제](#kubernetes-deployment-예제)
- [자동 계측 (Auto-Instrumentation)](#자동-계측-auto-instrumentation)
- [연동 확인 방법](#연동-확인-방법)
- [트러블슈팅](#트러블슈팅)

---

## 개요

```
┌──────────────────┐         OTLP (gRPC :4317)        ┌──────────────────────┐
│   Application    │ ──────────────────────────────▶   │   OTel Collector     │
│                  │         OTLP (HTTP :4318)         │   (DaemonSet)        │
│  ┌────────────┐  │ ──────────────────────────────▶   │                      │
│  │ OTel SDK   │  │                                   │  ┌─▶ VictoriaMetrics │
│  │            │  │   Traces ─────────────────────▶   │  ├─▶ Loki  (Logs)   │
│  │ • Traces   │  │   Metrics ────────────────────▶   │  └─▶ Tempo (Traces) │
│  │ • Metrics  │  │   Logs ───────────────────────▶   │                      │
│  │ • Logs     │  │                                   └──────────────────────┘
│  └────────────┘  │
└──────────────────┘
```

앱에서 OTel SDK를 초기화하면 **Traces, Metrics, Logs** 3가지 시그널을 모두 Collector로 전송할 수 있습니다.

---

## Collector 엔드포인트

Helm 릴리스 이름이 `observability`이고 네임스페이스가 `observability`인 경우:

| 프로토콜 | 엔드포인트 | 용도 |
|----------|-----------|------|
| **gRPC** | `observability-observability-stack-otel-collector.observability.svc.cluster.local:4317` | 고성능, 권장 |
| **HTTP** | `observability-observability-stack-otel-collector.observability.svc.cluster.local:4318` | gRPC 불가 시 |

> **DaemonSet 모드**에서는 각 노드에 hostPort `4317`, `4318`이 열려있으므로, 노드 IP를 통한 전송도 가능합니다.
>
> ```
> # 노드 IP 기반 (DaemonSet hostPort)
> gRPC: ${NODE_IP}:4317
> HTTP: ${NODE_IP}:4318
> ```

### 짧은 서비스명 (같은 네임스페이스 내)

같은 네임스페이스에 있다면 짧은 이름을 사용할 수 있습니다:

```
gRPC: observability-observability-stack-otel-collector:4317
HTTP: observability-observability-stack-otel-collector:4318
```

> 💡 아래 예제에서는 환경변수로 주입하므로, 실제 Helm 릴리스명에 맞게 변경하세요.

---

## 환경변수 설정

OpenTelemetry SDK는 공통 환경변수를 통해 설정할 수 있습니다. 모든 언어에서 동일하게 적용됩니다.

```yaml
env:
  # ── 기본 설정 ──
  - name: OTEL_SERVICE_NAME
    value: "my-app"                          # 서비스 이름 (필수)
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: "service.namespace=my-team,service.version=1.0.0,deployment.environment=production"

  # ── Collector 엔드포인트 (gRPC) ──
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://observability-observability-stack-otel-collector:4317"
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: "grpc"

  # ── 또는 DaemonSet hostPort 사용 시 ──
  # - name: NODE_IP
  #   valueFrom:
  #     fieldRef:
  #       fieldPath: status.hostIP
  # - name: OTEL_EXPORTER_OTLP_ENDPOINT
  #   value: "http://$(NODE_IP):4317"

  # ── 시그널별 개별 설정 (선택) ──
  # - name: OTEL_EXPORTER_OTLP_TRACES_ENDPOINT
  #   value: "http://observability-observability-stack-otel-collector:4317"
  # - name: OTEL_EXPORTER_OTLP_METRICS_ENDPOINT
  #   value: "http://observability-observability-stack-otel-collector:4317"
  # - name: OTEL_EXPORTER_OTLP_LOGS_ENDPOINT
  #   value: "http://observability-observability-stack-otel-collector:4317"

  # ── 샘플링 (선택) ──
  - name: OTEL_TRACES_SAMPLER
    value: "parentbased_traceidratio"
  - name: OTEL_TRACES_SAMPLER_ARG
    value: "1.0"                              # 1.0 = 100%, 0.1 = 10%

  # ── 전파 형식 ──
  - name: OTEL_PROPAGATORS
    value: "tracecontext,baggage"
```

---

## 언어별 SDK 연동

### Java (Spring Boot)

#### 방법 1: 자동 계측 (Zero-Code) — 권장

OTel Java Agent를 `-javaagent`로 주입하면 코드 변경 없이 자동 계측됩니다.

**Dockerfile:**

```dockerfile
FROM eclipse-temurin:21-jre

# OTel Java Agent 다운로드
ADD https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/download/v2.11.0/opentelemetry-javaagent.jar /opt/opentelemetry-javaagent.jar

COPY target/my-app.jar /app/my-app.jar

ENTRYPOINT ["java", \
  "-javaagent:/opt/opentelemetry-javaagent.jar", \
  "-jar", "/app/my-app.jar"]
```

**추가 환경변수 (Java 전용):**

```yaml
env:
  - name: OTEL_SERVICE_NAME
    value: "my-spring-app"
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://observability-observability-stack-otel-collector:4317"
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: "grpc"
  # Java Agent 추가 설정
  - name: OTEL_INSTRUMENTATION_JDBC_ENABLED
    value: "true"
  - name: OTEL_INSTRUMENTATION_SPRING_WEB_ENABLED
    value: "true"
  - name: OTEL_LOGS_EXPORTER
    value: "otlp"                    # 로그도 OTLP로 전송
```

#### 방법 2: SDK 수동 연동

**build.gradle:**

```groovy
dependencies {
    implementation platform('io.opentelemetry.instrumentation:opentelemetry-instrumentation-bom:2.11.0')
    implementation 'io.opentelemetry:opentelemetry-api'
    implementation 'io.opentelemetry:opentelemetry-sdk'
    implementation 'io.opentelemetry:opentelemetry-exporter-otlp'
    implementation 'io.opentelemetry:opentelemetry-sdk-extension-autoconfigure'
}
```

**Application 코드:**

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.api.metrics.Meter;
import io.opentelemetry.api.metrics.LongCounter;

// SDK는 환경변수(OTEL_*)로 자동 설정됨 (autoconfigure)
Tracer tracer = GlobalOpenTelemetry.getTracer("my-app");
Meter meter = GlobalOpenTelemetry.getMeter("my-app");

LongCounter requestCounter = meter
    .counterBuilder("http.server.request.count")
    .setDescription("Total HTTP requests")
    .build();

// 트레이스 생성
Span span = tracer.spanBuilder("processOrder").startSpan();
try (var scope = span.makeCurrent()) {
    requestCounter.add(1);
    // 비즈니스 로직
} catch (Exception e) {
    span.recordException(e);
    throw e;
} finally {
    span.end();
}
```

---

## Kubernetes Deployment 예제

실제 Kubernetes에 배포할 때의 전체 매니페스트 예제입니다.

### gRPC 엔드포인트 사용 (권장)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app.kubernetes.io/name: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: my-app
  template:
    metadata:
      labels:
        app.kubernetes.io/name: my-app
    spec:
      containers:
        - name: my-app
          image: my-registry/my-app:latest
          ports:
            - containerPort: 8080
          env:
            # ── OTel 공통 설정 ──
            - name: OTEL_SERVICE_NAME
              value: "my-app"
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://observability-observability-stack-otel-collector.observability.svc.cluster.local:4317"
            - name: OTEL_EXPORTER_OTLP_PROTOCOL
              value: "grpc"
            - name: OTEL_RESOURCE_ATTRIBUTES
              value: "service.namespace=my-team,service.version=1.0.0,deployment.environment=production"
            - name: OTEL_TRACES_SAMPLER
              value: "parentbased_traceidratio"
            - name: OTEL_TRACES_SAMPLER_ARG
              value: "1.0"
            - name: OTEL_PROPAGATORS
              value: "tracecontext,baggage"
            - name: OTEL_LOGS_EXPORTER
              value: "otlp"
            # ── Pod 메타데이터 (리소스 속성) ──
            - name: OTEL_RESOURCE_ATTRIBUTES_POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: OTEL_RESOURCE_ATTRIBUTES_NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
```

### DaemonSet hostPort 사용 (노드 로컬 전송)

네트워크 홉을 줄이고 싶을 때, 같은 노드의 Collector로 직접 전송합니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
        - name: my-app
          image: my-registry/my-app:latest
          env:
            - name: OTEL_SERVICE_NAME
              value: "my-app"
            # 노드 IP를 환경변수로 주입
            - name: NODE_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.hostIP
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://$(NODE_IP):4317"
            - name: OTEL_EXPORTER_OTLP_PROTOCOL
              value: "grpc"
```

> ⚡ **DaemonSet + hostPort 방식의 장점:**
> - 네트워크 지연 최소화 (같은 노드 내 통신)
> - Service DNS 의존 없음
> - Collector 장애 시 영향 범위가 해당 노드로 제한

---

## 자동 계측 (Auto-Instrumentation)

코드 변경 없이 자동으로 계측하는 방법입니다.

### 언어별 자동 계측 요약

| 언어 | 방법 | 코드 변경 필요 |
|------|------|:--------------:|
| **Java** | `-javaagent` JVM 옵션 | ❌ |
| **Python** | `opentelemetry-instrument` CLI | ❌ |
| **Node.js** | `--require ./tracing.js` | 초기화 파일 1개 |
| **Go** | 수동 SDK 연동 필수 | ✅ |
| **.NET** | `DOTNET_STARTUP_HOOKS` 환경변수 | ❌ |

### Java Agent 자동 계측으로 지원되는 라이브러리 (주요)

| 카테고리 | 라이브러리 |
|----------|-----------|
| HTTP 서버 | Spring Web MVC, Spring WebFlux, Servlet, Netty |
| HTTP 클라이언트 | RestTemplate, WebClient, OkHttp, Apache HttpClient |
| 데이터베이스 | JDBC, Hibernate, R2DBC, MongoDB, Redis (Jedis/Lettuce) |
| 메시징 | Kafka, RabbitMQ, gRPC |
| 캐시 | Caffeine, Ehcache |
| 로깅 | Logback, Log4j2 (자동 로그-트레이스 상관관계) |

---

## 연동 확인 방법

### 1. OTel Collector zPages 확인

Collector의 zPages에서 파이프라인 상태를 확인할 수 있습니다:

```bash
# 포트 포워딩
kubectl port-forward svc/observability-observability-stack-otel-collector -n observability 55679:55679

# 브라우저에서 확인
# http://localhost:55679/debug/tracez    → 트레이스 확인
# http://localhost:55679/debug/pipelinez → 파이프라인 상태
```

### 2. Grafana에서 확인

```bash
kubectl port-forward svc/observability-observability-stack-grafana -n observability 3000:3000
```

- **Traces**: Explore → Tempo 데이터소스 → 서비스명으로 검색
- **Metrics**: Explore → VictoriaMetrics 데이터소스 → `{service_name="my-app"}` 쿼리
- **Logs**: Explore → Loki 데이터소스 → `{service_name="my-app"}` 쿼리

### 3. 터미널에서 직접 테스트

```bash
# Collector에 테스트 트레이스 전송
kubectl run otel-test --rm -it --restart=Never \
  --image=ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:v0.115.1 \
  -n observability \
  -- traces \
  --otlp-endpoint observability-observability-stack-otel-collector:4317 \
  --otlp-insecure \
  --service my-test-service \
  --traces 5
```

### 4. Collector 로그 확인

```bash
# Collector 로그에서 수신된 데이터 확인
kubectl logs -l app.kubernetes.io/name=otel-collector -n observability --tail=50

# debug exporter 활성화 시 상세 로그 확인 가능
# values에서 otelCollector.config.debug: true 설정
```

---

## 트러블슈팅

### 데이터가 전송되지 않을 때

| 증상 | 확인사항 | 해결 |
|------|---------|------|
| Connection refused | Collector 서비스 확인 | `kubectl get svc -n observability` |
| DNS 오류 | 서비스명/네임스페이스 확인 | FQDN 사용: `svc.cluster.local` |
| 데이터 유실 | Collector 메모리 부족 | `resources.limits.memory` 증가 |
| 트레이스만 없음 | Tempo 연결 확인 | Collector 로그에서 exporter 에러 확인 |
| 메트릭만 없음 | VictoriaMetrics 연결 확인 | `prometheusremotewrite` exporter 상태 확인 |

### 유용한 디버깅 명령어

```bash
# Collector Pod 상태 확인
kubectl get pods -l app.kubernetes.io/name=otel-collector -n observability

# Collector 엔드포인트 연결 테스트
kubectl run curl-test --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s http://observability-observability-stack-otel-collector.observability:13133

# Collector 메트릭 확인 (수신/전송 카운터)
kubectl run curl-test --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s http://observability-observability-stack-otel-collector.observability:8888/metrics \
  | grep otelcol_receiver_accepted

# Tempo 연결 확인
kubectl run curl-test --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s http://observability-observability-stack-tempo.observability:3200/ready

# Loki 연결 확인
kubectl run curl-test --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s http://observability-observability-stack-loki.observability:3100/ready

# VictoriaMetrics 연결 확인
kubectl run curl-test --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s http://observability-observability-stack-vm.observability:8428/health
```

### 샘플링 조정 (트레이스 양이 너무 많을 때)

```yaml
env:
  - name: OTEL_TRACES_SAMPLER
    value: "parentbased_traceidratio"
  - name: OTEL_TRACES_SAMPLER_ARG
    value: "0.1"   # 10%만 샘플링 (프로덕션 권장)
```

> ⚠️ 메트릭과 로그는 샘플링되지 않습니다. 트레이스만 샘플링 대상입니다.
