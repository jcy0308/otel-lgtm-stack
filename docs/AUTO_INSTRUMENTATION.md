# 🔬 자동 계측 (Auto-Instrumentation) 원리

이 문서는 OpenTelemetry Java Agent의 자동 계측이 **어떻게 코드 변경 없이 로그, 메트릭, 트레이스를 수집하는지** 내부 원리를 설명합니다.

---

## 목차

- [개요](#개요)
- [Java Agent 동작 원리](#java-agent-동작-원리)
- [로그 자동 계측](#로그-자동-계측)
  - [바이트코드 변환 과정](#바이트코드-변환-과정)
  - [트레이스-로그 상관관계](#트레이스-로그-상관관계)
  - [로그 전송 경로](#로그-전송-경로)
- [메트릭 자동 계측](#메트릭-자동-계측)
  - [경로 A: Micrometer Bridge](#경로-a-micrometer-bridge)
  - [경로 B: Agent 직접 계측](#경로-b-agent-직접-계측)
  - [자동 수집되는 메트릭 목록](#자동-수집되는-메트릭-목록)
- [트레이스 자동 계측](#트레이스-자동-계측)
- [전체 아키텍처](#전체-아키텍처)
- [Python / Node.js 자동 계측](#python--nodejs-자동-계측)

---

## 개요

OpenTelemetry 자동 계측은 **애플리케이션 코드를 수정하지 않고** 텔레메트리 데이터를 수집하는 기술입니다.

```
개발자가 하는 것:
  -javaagent:opentelemetry-javaagent.jar  ← 이 한 줄만 추가

Agent가 하는 것:
  ✅ HTTP 요청/응답 트레이스
  ✅ DB 쿼리 트레이스 + 메트릭
  ✅ 로그에 trace_id 자동 삽입
  ✅ JVM/Spring 메트릭 수집
  ✅ Kafka/Redis/gRPC 자동 계측
  ✅ OTLP로 Collector에 전송
```

---

## Java Agent 동작 원리

Java Agent는 **JVM의 `java.lang.instrument` API**와 **ByteBuddy** 라이브러리를 사용하여 클래스 로딩 시점에 바이트코드를 변환합니다.

```
JVM 시작
  │
  ├─ 1. -javaagent:opentelemetry-javaagent.jar 로드
  │     └─ premain() 메서드 실행
  │
  ├─ 2. Agent가 InstrumentationModule 목록 스캔
  │     ├─ LogbackInstrumentationModule
  │     ├─ Log4j2InstrumentationModule
  │     ├─ ServletInstrumentationModule
  │     ├─ SpringWebMvcInstrumentationModule
  │     ├─ JdbcInstrumentationModule
  │     ├─ KafkaInstrumentationModule
  │     └─ ... (200+ 모듈)
  │
  ├─ 3. ClassFileTransformer 등록
  │     └─ 대상 클래스가 로딩될 때 바이트코드 변환
  │
  ├─ 4. 클래스 로딩 시 ByteBuddy로 바이트코드 변환
  │     예: ch.qos.logback.classic.Logger
  │         javax.servlet.http.HttpServlet
  │         java.sql.Statement
  │
  └─ 5. 변환된 클래스가 JVM에 로드됨
        └─ 원본 동작 + OTel 계측 코드가 함께 실행
```

> **핵심:** 소스코드는 변경되지 않습니다. JVM이 `.class` 파일을 메모리에 로드하는 순간에 바이트코드를 변환하는 것입니다.

---

## 로그 자동 계측

### 바이트코드 변환 과정

Agent는 Logback/Log4j2의 로그 출력 메서드를 감지하여 바이트코드를 변환합니다.

```java
// ── 원래 Logback 코드 ──
public class Logger {
    public void info(String msg) {
        // 로그 출력 (콘솔, 파일 등)
    }
}

// ── Agent가 바이트코드 변환 후 (개념적) ──
public class Logger {
    public void info(String msg) {
        // ── OTel 삽입 코드 시작 ──
        LogRecord record = LogRecord.builder()
            .setBody(msg)
            .setSeverity(Severity.INFO)
            .setTimestamp(Instant.now())
            .setContext(Span.current().getSpanContext())  // ← trace_id, span_id 자동 주입
            .build();
        logEmitter.emit(record);
        // ── OTel 삽입 코드 끝 ──

        // ── 원본 Logback 코드 (그대로 유지) ──
        // 로그 출력 (콘솔, 파일 등)
    }
}
```

### 트레이스-로그 상관관계

가장 강력한 기능은 **로그에 `trace_id`와 `span_id`가 자동 삽입**되는 것입니다.

```java
// 개발자가 작성한 코드
log.info("주문 처리 완료: orderId={}", orderId);

// 실제 OTel로 전송되는 LogRecord
{
  "body": "주문 처리 완료: orderId=12345",
  "severity": "INFO",
  "timestamp": "2026-03-11T10:30:00Z",
  "traceId": "abc123def456...",        // ← 자동 삽입
  "spanId": "789xyz...",               // ← 자동 삽입
  "resource": {
    "service.name": "order-service"
  }
}
```

이로 인해 Grafana에서:
- **트레이스 → 로그**: 특정 트레이스의 모든 관련 로그 조회 가능
- **로그 → 트레이스**: 에러 로그에서 해당 요청의 전체 트레이스로 이동 가능

### 로그 전송 경로

```
App: logger.info("주문 처리 완료")
  │
  ├─ 1) 기존 경로 (변경 없음)
  │     Logback Appender → 콘솔/파일 출력
  │
  └─ 2) OTel 경로 (Agent가 추가)
        Agent 삽입 코드 → OTel LogRecord 생성 (trace_id 포함)
          → BatchLogRecordProcessor (버퍼링, 비동기)
          → OtlpGrpcLogExporter
          → Collector :4317 (OTLP gRPC)
          → Loki
```

> 기존 로그 출력은 **그대로 유지**됩니다. 콘솔에도 파일에도 정상 출력되면서, 동시에 OTLP로도 전송됩니다.

---

## 메트릭 자동 계측

Spring Boot 메트릭은 **두 가지 경로**로 수집됩니다.

### 경로 A: Micrometer Bridge

Spring Boot는 **Micrometer**라는 메트릭 추상화 라이브러리를 내장하고 있습니다. OTel Agent는 Micrometer의 `MeterRegistry`에 **`OpenTelemetryMeterRegistry`를 Bridge로 주입**합니다.

```
Spring Boot 내부
  │
  ├─ Micrometer MeterRegistry (Spring이 이미 만들어 둔 것)
  │     ├─ http.server.requests      ← Spring WebMVC가 자동 등록
  │     ├─ jvm.memory.used           ← JvmMemoryMetrics가 자동 등록
  │     ├─ jvm.gc.pause              ← JvmGcMetrics가 자동 등록
  │     ├─ hikaricp.connections       ← HikariCP가 자동 등록
  │     ├─ process.cpu.usage         ← ProcessorMetrics가 자동 등록
  │     └─ ...
  │
  ├─ OTel Agent가 OpenTelemetryMeterRegistry 주입
  │     └─ Micrometer 메트릭 → OTel Metric으로 변환
  │
  └─ OTel SDK
        → PeriodicMetricReader (30초 간격)
        → OtlpGrpcMetricExporter
        → Collector :4317
        → VictoriaMetrics
```

**Micrometer가 메트릭을 만드는 예시 (Spring 내부 코드):**

```java
// Spring WebMVC의 WebMvcMetricsFilter (Spring 내장)
public class WebMvcMetricsFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, ...) {
        Timer.Sample sample = Timer.start(meterRegistry);
        try {
            filterChain.doFilter(request, response);
        } finally {
            // http.server.requests 타이머에 기록
            sample.stop(Timer.builder("http.server.requests")
                .tag("method", request.getMethod())
                .tag("uri", request.getRequestURI())
                .tag("status", String.valueOf(response.getStatus()))
                .register(meterRegistry));
        }
    }
}
// → OTel Agent가 이 meterRegistry를 Bridge로 연결
//   → 이 메트릭이 OTel SDK → Collector → VictoriaMetrics로 전송
```

### 경로 B: Agent 직접 계측

Agent가 Micrometer를 거치지 않고 **직접 바이트코드를 변환**하여 메트릭을 생성하는 경우도 있습니다.

```
JDBC 호출 (java.sql.Statement.execute())
  │
  ├─ Agent가 바이트코드 변환
  │     ├─ Span 생성 (트레이스)
  │     └─ db.client.connections.usage 메트릭 직접 기록 (메트릭)
  │
  └─ OTel SDK → Collector → VictoriaMetrics
```

```
HTTP Client 호출 (RestTemplate, OkHttp 등)
  │
  ├─ Agent가 바이트코드 변환
  │     ├─ Span 생성 (트레이스)
  │     └─ http.client.request.duration 메트릭 직접 기록 (메트릭)
  │
  └─ OTel SDK → Collector → VictoriaMetrics
```

### 자동 수집되는 메트릭 목록

#### JVM 메트릭 (Micrometer Bridge 경로)

| 메트릭 | 설명 | 타입 |
|--------|------|------|
| `jvm.memory.used` | 힙/논힙 메모리 사용량 | Gauge |
| `jvm.memory.committed` | JVM에 할당된 메모리 | Gauge |
| `jvm.memory.max` | 최대 메모리 | Gauge |
| `jvm.gc.duration` | GC 소요 시간 | Histogram |
| `jvm.gc.count` | GC 실행 횟수 | Counter |
| `jvm.gc.memory.promoted` | Old Gen으로 프로모션된 바이트 | Counter |
| `jvm.thread.count` | 활성 스레드 수 | Gauge |
| `jvm.thread.daemon` | 데몬 스레드 수 | Gauge |
| `jvm.class.loaded` | 현재 로딩된 클래스 수 | Gauge |
| `jvm.buffer.memory.used` | 버퍼풀 메모리 사용량 | Gauge |
| `process.runtime.jvm.cpu.utilization` | JVM CPU 사용률 | Gauge |

#### Spring HTTP 메트릭 (Micrometer Bridge 경로)

| 메트릭 | 설명 | 태그 |
|--------|------|------|
| `http.server.request.duration` | 요청 처리 시간 | method, uri, status |
| `http.server.active_requests` | 현재 처리 중인 요청 수 | method |

#### DB 커넥션풀 메트릭 (Micrometer Bridge 경로)

| 메트릭 | 설명 |
|--------|------|
| `db.client.connections.usage` | 커넥션 상태 (active/idle) |
| `db.client.connections.max` | 최대 커넥션 수 |
| `db.client.connections.pending_requests` | 커넥션 대기 중인 요청 수 |
| `db.client.connections.create_time` | 커넥션 생성 시간 |

#### Agent 직접 계측 메트릭 (경로 B)

| 메트릭 | 설명 | 계측 대상 |
|--------|------|----------|
| `http.client.request.duration` | HTTP 클라이언트 요청 시간 | RestTemplate, WebClient, OkHttp |
| `db.client.operation.duration` | DB 쿼리 실행 시간 | JDBC |
| `messaging.process.duration` | 메시지 처리 시간 | Kafka Consumer |
| `messaging.publish.duration` | 메시지 발행 시간 | Kafka Producer |
| `rpc.client.duration` | gRPC 클라이언트 호출 시간 | gRPC |
| `rpc.server.duration` | gRPC 서버 처리 시간 | gRPC |

---

## 트레이스 자동 계측

Agent는 프레임워크의 핵심 클래스에 Span 생성 코드를 삽입합니다.

```java
// ── 원래 Servlet 코드 ──
public class HttpServlet {
    protected void service(HttpServletRequest req, HttpServletResponse resp) {
        // 요청 처리
    }
}

// ── Agent 변환 후 (개념적) ──
public class HttpServlet {
    protected void service(HttpServletRequest req, HttpServletResponse resp) {
        // ── OTel Span 시작 ──
        Span span = tracer.spanBuilder("HTTP " + req.getMethod())
            .setSpanKind(SpanKind.SERVER)
            .setAttribute("http.method", req.getMethod())
            .setAttribute("http.url", req.getRequestURL().toString())
            .setAttribute("http.scheme", req.getScheme())
            .startSpan();
        try (Scope scope = span.makeCurrent()) {
            // ── 원본 코드 실행 ──
            // 요청 처리

            span.setAttribute("http.status_code", resp.getStatus());
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR);
            throw e;
        } finally {
            span.end();
            // ── OTel Span 종료 ──
        }
    }
}
```

### 자동 생성되는 Span 체인 예시

```
[HTTP GET /api/orders/123]                          (Spring WebMVC)
  ├─ [OrderController.getOrder]                     (Spring AOP)
  │   ├─ [SELECT * FROM orders WHERE id=?]          (JDBC)
  │   ├─ [Redis GET order:123:cache]                (Lettuce)
  │   └─ [HTTP GET http://payment-svc/api/pay/123]  (RestTemplate)
  │       └─ [전파된 Span - payment-service]        (W3C TraceContext)
  └─ [HTTP Response 200]
```

### Context Propagation (서비스 간 전파)

```
┌─ Order Service ──────────────┐     ┌─ Payment Service ─────────────┐
│                               │     │                                │
│  Span A (HTTP Server)         │     │  Span B (HTTP Server)          │
│    │                          │     │    ↑                           │
│    └─ Span C (HTTP Client) ──────────── traceparent 헤더 전파        │
│         RestTemplate.get()    │     │    Agent가 자동으로             │
│                               │     │    부모 Span으로 연결          │
└───────────────────────────────┘     └────────────────────────────────┘

HTTP 헤더:
  traceparent: 00-abc123def456...-789xyz...-01
  ↑ W3C TraceContext 표준 (Agent가 자동으로 삽입/추출)
```

---

## 전체 아키텍처

```
┌─ Spring Boot Application ─────────────────────────────────────────────┐
│                                                                        │
│  ┌─ OTel Java Agent (바이트코드 변환) ─────────────────────────────┐  │
│  │                                                                  │  │
│  │  [로그]                                                          │  │
│  │  Logback/Log4j2                                                  │  │
│  │    → LogRecord 생성 + trace_id 자동 삽입 ─────────┐             │  │
│  │                                                    │             │  │
│  │  [트레이스]                                        │             │  │
│  │  Servlet/WebFlux/RestTemplate/JDBC/Kafka           │             │  │
│  │    → Span 생성 + Context Propagation ──────────────┤             │  │
│  │                                                    │  OTLP gRPC  │  │
│  │  [메트릭 - 경로B: Agent 직접]                     ├─────────────────▶ OTel
│  │  JDBC/HTTP Client/Kafka                            │             │  │ Collector
│  │    → 직접 계측하여 Metric 기록 ───────────────────┤             │  │  :4317
│  │                                                    │             │  │
│  │  [메트릭 - 경로A: Micrometer Bridge]              │             │  │
│  │  Spring Actuator + Micrometer MeterRegistry        │             │  │
│  │    ├─ http.server.requests                         │             │  │
│  │    ├─ jvm.memory.*                                 │             │  │
│  │    ├─ jvm.gc.*              → OTel Bridge ─────────┘             │  │
│  │    ├─ hikaricp.*                                                 │  │
│  │    └─ process.runtime.*                                          │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  [기존 동작 유지]                                                      │
│    └─ Logback → 콘솔/파일 출력 (변경 없음)                            │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Python / Node.js 자동 계측

### Python

Python은 `opentelemetry-instrument` CLI가 **monkey patching**으로 라이브러리 함수를 래핑합니다.

```
opentelemetry-instrument python app.py
  │
  ├─ sitecustomize.py에서 자동 계측 초기화
  ├─ 감지된 라이브러리의 함수를 래핑 (monkey patch)
  │     ├─ flask.Flask.wsgi_app → 트레이스 래핑
  │     ├─ psycopg2.connect → DB 트레이스 래핑
  │     ├─ requests.Session.send → HTTP 클라이언트 래핑
  │     └─ logging.Logger.handle → 로그에 trace_id 삽입
  └─ OTLP로 Collector에 전송
```

Java의 바이트코드 변환과 달리, Python은 **런타임에 함수 참조를 교체**하는 방식입니다.

### Node.js

Node.js는 `require` 훅으로 모듈 로딩을 가로채 래핑합니다.

```
node --require ./tracing.js app.js
  │
  ├─ tracing.js에서 NodeSDK 초기화
  ├─ auto-instrumentations-node가 require 훅 등록
  │     ├─ http/https 모듈 → 요청/응답 트레이스
  │     ├─ express 라우터 → 라우트별 Span
  │     ├─ pg (PostgreSQL) → 쿼리 트레이스
  │     └─ ioredis → Redis 명령 트레이스
  └─ OTLP로 Collector에 전송
```

### 언어별 자동 계측 비교

| 항목 | Java | Python | Node.js | Go |
|------|------|--------|---------|-----|
| **방식** | 바이트코드 변환 | Monkey patching | require 훅 | 불가 (수동만) |
| **성능 오버헤드** | 매우 낮음 (~2%) | 낮음 (~3%) | 낮음 (~3%) | N/A |
| **코드 변경** | 없음 | 없음 | 초기화 파일 1개 | 전체 수동 |
| **적용 시점** | JVM 시작 시 | 프로세스 시작 시 | 모듈 로딩 시 | 컴파일 시 |
| **지원 라이브러리** | 200+ | 40+ | 30+ | 10+ |
