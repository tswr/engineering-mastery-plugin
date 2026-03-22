# Java Observability Reference

Idiomatic Java patterns for observability. Uses SLF4J with Logback for structured logging (MDC for context), Micrometer for metrics, and the OpenTelemetry Java SDK for distributed tracing. JVM debugging tools include jstack, jmap, and JDK Flight Recorder.

---

## Structured Logging

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
import static net.logstash.logback.argument.StructuredArguments.kv;

private static final Logger log = LoggerFactory.getLogger(OrderService.class);

// Structured key-value pairs via logstash-logback-encoder
log.info("order processed", kv("orderId", order.getId()),
         kv("total", order.getTotal()), kv("items", order.getItems().size()));
log.error("payment failed", kv("orderId", order.getId()), kv("error", e.getMessage()));

// MDC: thread-local context propagated to all log statements
public void handleRequest(HttpServletRequest request) {
    MDC.put("requestId", request.getHeader("X-Request-ID"));
    MDC.put("userId", getCurrentUserId());
    try {
        log.info("request started", kv("method", request.getMethod()));
        processRequest(request);  // all logs include requestId + userId
    } finally {
        MDC.clear();  // prevent context leaking across requests
    }
}

// logback.xml: use LogstashEncoder for JSON output
// <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
```

## Metrics and Instrumentation

```java
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.Timer;
import io.micrometer.core.instrument.Gauge;
import io.micrometer.core.instrument.MeterRegistry;
import java.util.concurrent.atomic.AtomicInteger;

public class OrderHandler {
    private final Counter requestCounter;
    private final Timer requestDuration;
    private final AtomicInteger inProgress = new AtomicInteger(0);

    public OrderHandler(MeterRegistry registry) {
        this.requestCounter = Counter.builder("http.requests.total")
            .tag("endpoint", "/orders").register(registry);
        this.requestDuration = Timer.builder("http.request.duration")
            .publishPercentiles(0.5, 0.95, 0.99)
            .publishPercentileHistogram()
            .register(registry);
        Gauge.builder("http.requests.in_progress", inProgress, AtomicInteger::get)
            .register(registry);
    }

    public Response handle(Request request) {
        inProgress.incrementAndGet();
        return requestDuration.record(() -> {
            try {
                Response resp = process(request);
                requestCounter.increment();
                return resp;
            } finally {
                inProgress.decrementAndGet();
            }
        });
    }
}
// Spring Boot: add micrometer-registry-prometheus for /actuator/prometheus
```

## Distributed Tracing

```java
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.context.Scope;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;
import io.opentelemetry.sdk.resources.Resource;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.semconv.ServiceAttributes;

public static OpenTelemetrySdk initTracing() {
    Resource resource = Resource.getDefault()
        .merge(Resource.create(Attributes.of(ServiceAttributes.SERVICE_NAME, "order-service")));
    SdkTracerProvider provider = SdkTracerProvider.builder()
        .addSpanProcessor(BatchSpanProcessor.builder(
            OtlpGrpcSpanExporter.builder().build()).build())
        .setResource(resource).build();
    return OpenTelemetrySdk.builder().setTracerProvider(provider).buildAndRegisterGlobal();
}

public Receipt processOrder(Order order) {
    Span span = tracer.spanBuilder("processOrder").startSpan();
    try (Scope scope = span.makeCurrent()) {
        span.setAttribute("order.id", order.getId());
        span.setAttribute("order.item_count", order.getItems().size());
        validateOrder(order);
        chargePayment(order);
        span.addEvent("payment_charged",
            Attributes.of(AttributeKey.stringKey("provider"), "stripe"));
        return fulfill(order);
    } catch (Exception e) {
        span.setStatus(StatusCode.ERROR, e.getMessage());
        span.recordException(e);
        throw e;
    } finally {
        span.end();
    }
}
```

## Alerting

```java
// Expose SLI-friendly metrics; alerting rules live in Prometheus config.
private final Counter totalRequests = Counter.builder("http.requests.total")
    .tag("type", "all").register(registry);
private final Counter successRequests = Counter.builder("http.requests.total")
    .tag("type", "success").register(registry);

public Response handle(Request req) {
    totalRequests.increment();
    try {
        Response resp = process(req);
        successRequests.increment();  // only on success — enables SLO ratio
        return resp;
    } catch (Exception e) {
        log.error("request failed", kv("error", e.getMessage()));
        throw e;
    }
}
```

## Systematic Debugging

```java
// jstack: thread dump of a running JVM
// $ jstack <pid>                  # all threads
// $ jstack -l <pid>               # include lock info

// jmap: heap inspection
// $ jmap -histo <pid>             # object histogram
// $ jmap -dump:format=b,file=heap.hprof <pid>

// JDK Flight Recorder: low-overhead production profiling
// $ java -XX:StartFlightRecording=duration=60s,filename=rec.jfr -jar app.jar
// $ jcmd <pid> JFR.start duration=60s filename=rec.jfr  # attach at runtime

// Per-package log levels in logback.xml
// <logger name="com.example.payment" level="DEBUG"/>
// <root level="WARN"><appender-ref ref="STDOUT"/></root>

// Guard expensive debug logging
if (log.isDebugEnabled()) {
    log.debug("processing payment", kv("details", payment.toDetailedString()));
}

// Remote debugging
// $ java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005 -jar app.jar
```

## Dashboards and Operational Readiness

```java
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;

public class ServiceHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        boolean dbOk = checkDatabase();
        boolean cacheOk = checkRedis();
        var builder = (dbOk && cacheOk) ? Health.up() : Health.down();
        return builder.withDetail("database", dbOk).withDetail("cache", cacheOk).build();
    }
}

public void verifyObservability(OpenTelemetry otel) {
    if (otel.getTracerProvider() == null) {
        throw new IllegalStateException("Tracer provider not configured");
    }
    log.info("observability ready", kv("tracing", true), kv("metrics", true));
}
```
