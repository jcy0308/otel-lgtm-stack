# 📊 커스텀 메트릭 작성 가이드

이 문서는 OpenTelemetry SDK를 사용하여 **비즈니스 로직에 맞는 커스텀 메트릭**을 작성하는 방법을 설명합니다.

자동 계측이 JVM/HTTP 등 인프라 메트릭을 수집한다면, 커스텀 메트릭은 **주문 수, 결제 금액, 재고량** 같은 비즈니스 지표를 수집합니다.

---

## 목차

- [메트릭 타입 개요](#메트릭-타입-개요)
- [어떤 메트릭 타입을 써야 하나?](#어떤-메트릭-타입을-써야-하나)
- [Java (Spring Boot)](#java-spring-boot)
  - [설정](#java-설정)
  - [Counter 예제](#java-counter-예제)
  - [Histogram 예제](#java-histogram-예제)
  - [Gauge 예제](#java-gauge-예제)
  - [UpDownCounter 예제](#java-updowncounter-예제)
  - [실전: 주문 서비스 예제](#java-실전-주문-서비스-예제)
- [메트릭 네이밍 컨벤션](#메트릭-네이밍-컨벤션)
- [Attribute (태그) 설계 가이드](#attribute-태그-설계-가이드)
- [Grafana 대시보드에서 조회](#grafana-대시보드에서-조회)

---

## 메트릭 타입 개요

OpenTelemetry는 4가지 메트릭 타입을 제공합니다.

| 타입 | 설명 | 예시 | Prometheus 대응 |
|------|------|------|----------------|
| **Counter** | 누적 증가만 하는 값 | 총 요청 수, 총 에러 수 | Counter |
| **UpDownCounter** | 증가/감소하는 값 | 현재 활성 연결 수, 큐 크기 | Gauge |
| **Histogram** | 값의 분포 측정 | 응답 시간, 요청 크기 | Histogram |
| **Gauge** | 특정 시점의 값 (비동기) | CPU 온도, 메모리 사용량, 재고량 | Gauge |

```
Counter:        0 → 1 → 2 → 3 → 4 → 5          (단조 증가)
UpDownCounter:  0 → 1 → 3 → 2 → 4 → 1          (증감 가능)
Histogram:      [12ms, 45ms, 3ms, 120ms, 8ms]   (값 분포 기록)
Gauge:          72.5 → 68.3 → 75.1 → 69.8       (시점 값, 콜백으로 수집)
```

---

## 어떤 메트릭 타입을 써야 하나?

| 측정하고 싶은 것 | 메트릭 타입 | 예시 |
|-----------------|-----------|------|
| "총 몇 번 발생했나?" | **Counter** | 주문 건수, 에러 횟수, 로그인 횟수 |
| "현재 몇 개인가?" | **UpDownCounter** | 활성 세션 수, 큐에 쌓인 작업 수 |
| "얼마나 걸렸나? / 얼마나 큰가?" | **Histogram** | 응답 시간, 결제 금액, 파일 크기 |
| "현재 상태값은?" | **Gauge** | 캐시 적중률, 재고량, 외부 API 응답시간 |

---

## Java (Spring Boot)

### Java 설정

OTel Java Agent를 사용하면 SDK가 자동 초기화됩니다. 코드에서 `GlobalOpenTelemetry`를 통해 바로 사용할 수 있습니다.

```xml
<!-- pom.xml (API 의존성만 추가 — SDK는 Agent가 제공) -->
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-api</artifactId>
</dependency>

<!-- BOM으로 버전 관리 -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.opentelemetry</groupId>
            <artifactId>opentelemetry-bom</artifactId>
            <version>1.44.1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

```groovy
// build.gradle
implementation platform('io.opentelemetry:opentelemetry-bom:1.44.1')
implementation 'io.opentelemetry:opentelemetry-api'
```

> ⚠️ `opentelemetry-api`만 추가하세요. `-sdk`, `-exporter` 등은 Agent가 제공하므로 중복 추가하면 충돌합니다.

### Java Counter 예제

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.LongCounter;
import io.opentelemetry.api.metrics.Meter;
import org.springframework.stereotype.Component;

@Component
public class OrderMetrics {

    private final LongCounter orderCounter;
    private final LongCounter orderErrorCounter;

    public OrderMetrics() {
        Meter meter = GlobalOpenTelemetry.getMeter("order-service");

        this.orderCounter = meter
            .counterBuilder("orders.created.total")         // 메트릭 이름
            .setDescription("Total number of orders created")
            .setUnit("{orders}")                              // 단위
            .build();

        this.orderErrorCounter = meter
            .counterBuilder("orders.errors.total")
            .setDescription("Total number of order processing errors")
            .setUnit("{errors}")
            .build();
    }

    public void recordOrderCreated(String productType, String region) {
        orderCounter.add(1, Attributes.builder()
            .put("order.product_type", productType)    // 상품 유형별 집계 가능
            .put("order.region", region)                // 지역별 집계 가능
            .build());
    }

    public void recordOrderError(String errorType) {
        orderErrorCounter.add(1, Attributes.builder()
            .put("error.type", errorType)
            .build());
    }
}
```

### Java Histogram 예제

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.DoubleHistogram;
import io.opentelemetry.api.metrics.Meter;
import org.springframework.stereotype.Component;

@Component
public class PaymentMetrics {

    private final DoubleHistogram paymentAmountHistogram;
    private final DoubleHistogram paymentDurationHistogram;

    public PaymentMetrics() {
        Meter meter = GlobalOpenTelemetry.getMeter("payment-service");

        // 결제 금액 분포
        this.paymentAmountHistogram = meter
            .histogramBuilder("payments.amount")
            .setDescription("Distribution of payment amounts")
            .setUnit("KRW")
            .build();

        // 결제 처리 시간 분포
        this.paymentDurationHistogram = meter
            .histogramBuilder("payments.processing.duration")
            .setDescription("Time taken to process payments")
            .setUnit("ms")
            .build();
    }

    public void recordPayment(double amount, String method, long durationMs) {
        Attributes attrs = Attributes.builder()
            .put("payment.method", method)  // "card", "bank_transfer", "kakao_pay"
            .build();

        paymentAmountHistogram.record(amount, attrs);
        paymentDurationHistogram.record(durationMs, attrs);
    }
}
```

### Java Gauge 예제

Gauge는 비동기(콜백) 방식으로 동작합니다. 주기적으로 호출되는 콜백 함수를 등록합니다.

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.Meter;
import org.springframework.stereotype.Component;
import jakarta.annotation.PostConstruct;

@Component
public class InventoryMetrics {

    private final InventoryRepository inventoryRepository;

    public InventoryMetrics(InventoryRepository inventoryRepository) {
        this.inventoryRepository = inventoryRepository;
    }

    @PostConstruct
    public void registerGauges() {
        Meter meter = GlobalOpenTelemetry.getMeter("inventory-service");

        // 재고량 — 콜백이 주기적으로 호출됨
        meter.gaugeBuilder("inventory.stock.level")
            .setDescription("Current stock level by product category")
            .setUnit("{items}")
            .buildWithCallback(measurement -> {
                // 카테고리별 재고량 조회
                Map<String, Long> stockByCategory = inventoryRepository.getStockByCategory();
                stockByCategory.forEach((category, count) -> {
                    measurement.record(count, Attributes.builder()
                        .put("product.category", category)
                        .build());
                });
            });

        // 캐시 적중률
        meter.gaugeBuilder("cache.hit.ratio")
            .setDescription("Cache hit ratio")
            .setUnit("1")                              // 비율은 단위 "1"
            .buildWithCallback(measurement -> {
                double hitRatio = cacheManager.getHitRatio();
                measurement.record(hitRatio);
            });
    }
}
```

### Java UpDownCounter 예제

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.LongUpDownCounter;
import io.opentelemetry.api.metrics.Meter;
import org.springframework.stereotype.Component;

@Component
public class QueueMetrics {

    private final LongUpDownCounter activeJobsCounter;

    public QueueMetrics() {
        Meter meter = GlobalOpenTelemetry.getMeter("worker-service");

        this.activeJobsCounter = meter
            .upDownCounterBuilder("jobs.active.count")
            .setDescription("Number of currently active jobs")
            .setUnit("{jobs}")
            .build();
    }

    public void onJobStarted(String jobType) {
        activeJobsCounter.add(1, Attributes.builder()
            .put("job.type", jobType)
            .build());
    }

    public void onJobCompleted(String jobType) {
        activeJobsCounter.add(-1, Attributes.builder()  // -1로 감소
            .put("job.type", jobType)
            .build());
    }
}
```

### Java 실전: 주문 서비스 예제

모든 메트릭 타입을 사용한 실전 예제입니다.

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.metrics.*;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.Tracer;
import org.springframework.stereotype.Service;

@Service
public class OrderService {

    private final Tracer tracer;
    private final LongCounter ordersCreated;
    private final LongCounter ordersFailed;
    private final DoubleHistogram orderAmount;
    private final DoubleHistogram orderProcessingTime;
    private final LongUpDownCounter ordersInProgress;

    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;

        Meter meter = GlobalOpenTelemetry.getMeter("order-service");
        this.tracer = GlobalOpenTelemetry.getTracer("order-service");

        // Counter: 주문 생성 건수
        this.ordersCreated = meter.counterBuilder("orders.created.total")
            .setDescription("Total orders created")
            .setUnit("{orders}")
            .build();

        // Counter: 주문 실패 건수
        this.ordersFailed = meter.counterBuilder("orders.failed.total")
            .setDescription("Total orders failed")
            .setUnit("{orders}")
            .build();

        // Histogram: 주문 금액 분포
        this.orderAmount = meter.histogramBuilder("orders.amount")
            .setDescription("Order amount distribution")
            .setUnit("KRW")
            .build();

        // Histogram: 주문 처리 시간
        this.orderProcessingTime = meter.histogramBuilder("orders.processing.duration")
            .setDescription("Order processing duration")
            .setUnit("ms")
            .build();

        // UpDownCounter: 현재 처리 중인 주문
        this.ordersInProgress = meter.upDownCounterBuilder("orders.in_progress")
            .setDescription("Orders currently being processed")
            .setUnit("{orders}")
            .build();

        // Gauge: 미처리 주문 수 (비동기 콜백)
        meter.gaugeBuilder("orders.pending.count")
            .setDescription("Number of pending orders")
            .setUnit("{orders}")
            .ofLongs()
            .buildWithCallback(measurement -> {
                long pendingCount = orderRepository.countByStatus("PENDING");
                measurement.record(pendingCount);
            });
    }

    public Order createOrder(OrderRequest request) {
        Attributes attrs = Attributes.builder()
            .put("order.product_type", request.getProductType())
            .put("order.region", request.getRegion())
            .put("order.channel", request.getChannel())   // "web", "mobile", "api"
            .build();

        // 처리 중 카운터 증가
        ordersInProgress.add(1, attrs);
        long startTime = System.currentTimeMillis();

        // 커스텀 Span 생성 (트레이스)
        Span span = tracer.spanBuilder("createOrder")
            .setAttribute("order.product_type", request.getProductType())
            .setAttribute("order.amount", request.getAmount())
            .startSpan();

        try (var scope = span.makeCurrent()) {
            Order order = orderRepository.save(request.toEntity());

            // 주문 성공 메트릭
            ordersCreated.add(1, attrs);
            orderAmount.record(request.getAmount(), attrs);

            long duration = System.currentTimeMillis() - startTime;
            orderProcessingTime.record(duration, attrs);

            span.setAttribute("order.id", order.getId());
            return order;

        } catch (Exception e) {
            // 주문 실패 메트릭
            ordersFailed.add(1, Attributes.builder()
                .put("order.product_type", request.getProductType())
                .put("error.type", e.getClass().getSimpleName())
                .build());

            span.recordException(e);
            throw e;
        } finally {
            ordersInProgress.add(-1, attrs);
            span.end();
        }
    }
}
```

---

## 메트릭 네이밍 컨벤션

### OTel Semantic Conventions 기반 규칙

```
<namespace>.<entity>.<action/state>
```

| 패턴 | 예시 | 설명 |
|------|------|------|
| `{namespace}.{entity}.total` | `orders.created.total` | 누적 카운터 |
| `{namespace}.{entity}.duration` | `orders.processing.duration` | 시간 히스토그램 |
| `{namespace}.{entity}.count` | `orders.pending.count` | 현재 값 (Gauge) |
| `{namespace}.{entity}.size` | `orders.payload.size` | 크기 히스토그램 |

### 좋은 예 / 나쁜 예

| ❌ 나쁜 예 | ✅ 좋은 예 | 이유 |
|-----------|-----------|------|
| `order_count` | `orders.created.total` | 네임스페이스 + 구체적인 동작 |
| `latency` | `http.server.request.duration` | 무엇의 지연인지 명확 |
| `errors` | `orders.errors.total` | 도메인 컨텍스트 포함 |
| `dbTime` | `db.client.operation.duration` | 카멜케이스 ❌, 점(.) 구분 |
| `cpu_percent` | `system.cpu.utilization` | Semantic Conventions 준수 |

### 단위 표기

| 단위 | 값 | 예시 |
|------|-----|------|
| 시간 | `s`, `ms`, `us`, `ns` | `orders.processing.duration` (ms) |
| 바이트 | `By`, `KiBy`, `MiBy` | `http.request.body.size` (By) |
| 개수 | `{items}`, `{requests}`, `{errors}` | `orders.created.total` ({orders}) |
| 비율 | `1` | `cache.hit.ratio` (0.0~1.0) |
| 통화 | `KRW`, `USD` | `payments.amount` (KRW) |

---

## Attribute (태그) 설계 가이드

### 카디널리티 주의

Attribute 값의 고유 조합 수(카디널리티)가 높으면 메트릭 저장 비용이 폭발합니다.

| ❌ 높은 카디널리티 (위험) | ✅ 낮은 카디널리티 (안전) |
|--------------------------|------------------------|
| `user.id` (수만~수백만) | `user.tier` (free/pro/enterprise) |
| `order.id` (무한 증가) | `order.product_type` (5~20종) |
| `request.url` (/users/12345) | `request.route` (/users/{id}) |
| `timestamp` | `hour_of_day` (0~23) |
| `ip.address` | `region` (kr/us/eu) |

### 권장 Attribute 패턴

```java
// ✅ 좋은 예: 낮은 카디널리티, 집계에 유용
Attributes.builder()
    .put("order.product_type", "electronics")   // ~20종
    .put("order.region", "kr")                   // ~10종
    .put("order.channel", "mobile")              // web/mobile/api
    .put("payment.method", "card")               // card/bank/kakao_pay
    .build();

// ❌ 나쁜 예: 높은 카디널리티
Attributes.builder()
    .put("order.id", "ORD-2026031100001")        // 무한 증가 → 절대 금지
    .put("user.id", "user-12345")                // 수만 개 → VictoriaMetrics 과부하
    .put("request.body", "{...}")                // 무한 종류 → 절대 금지
    .build();
```

> 💡 **경험 법칙:** 하나의 메트릭에서 attribute 값 조합이 **1,000개를 초과하면** 카디널리티가 너무 높은 것입니다. 설계를 재검토하세요.

---

## Grafana 대시보드에서 조회

커스텀 메트릭은 VictoriaMetrics에 저장되며, Grafana에서 PromQL로 조회합니다.

### Counter 조회

```promql
# 초당 주문 생성률 (rate)
rate(orders_created_total[5m])

# 상품 유형별 주문 생성률
sum by (order_product_type) (rate(orders_created_total[5m]))

# 최근 1시간 총 에러 수
increase(orders_errors_total[1h])
```

### Histogram 조회

```promql
# 평균 결제 금액
rate(payments_amount_sum[5m]) / rate(payments_amount_count[5m])

# 결제 처리 시간 p95
histogram_quantile(0.95, rate(payments_processing_duration_bucket[5m]))

# 결제 방법별 p99
histogram_quantile(0.99,
  sum by (le, payment_method) (rate(payments_processing_duration_bucket[5m]))
)
```

### Gauge 조회

```promql
# 현재 재고량
inventory_stock_level

# 카테고리별 재고량
inventory_stock_level{product_category="electronics"}

# 미처리 주문 수 추이
orders_pending_count
```

### UpDownCounter 조회

```promql
# 현재 처리 중인 주문 수
orders_in_progress

# 유형별 활성 작업 수
sum by (job_type) (jobs_active_count)
```

> ⚠️ **OTel → Prometheus 네이밍 변환:** OTel의 `orders.created.total`은 Prometheus에서 `orders_created_total`로 변환됩니다 (점 → 밑줄).
