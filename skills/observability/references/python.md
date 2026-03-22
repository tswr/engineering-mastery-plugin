# Python Observability Reference

Idiomatic Python patterns for observability. Uses `structlog` for structured logging, `prometheus_client` for metrics, and the `opentelemetry` SDK for distributed tracing.

---

## Structured Logging

```python
import structlog

# Configure structlog: JSON output, stdlib integration, context vars
structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ],
    wrapper_class=structlog.stdlib.BoundLogger,
    logger_factory=structlog.stdlib.LoggerFactory(),
)
logger = structlog.get_logger()

# Structured key-value pairs — never format strings manually
logger.info("order_processed", order_id=order.id, total=order.total, items=len(order.items))
logger.error("payment_failed", order_id=order.id, error=str(exc), provider="stripe")

# Bind context for the lifetime of a request
log = logger.bind(request_id=request_id, user_id=user.id)
log.info("request_started", method="POST", path="/orders")

# Context variables: auto-attach fields in async/threaded code
from structlog.contextvars import bind_contextvars, clear_contextvars

def middleware(request, call_next):
    clear_contextvars()
    bind_contextvars(request_id=request.headers["X-Request-ID"])
    return call_next(request)
    # all log calls within this request now include request_id
```

## Metrics and Instrumentation

```python
from prometheus_client import Counter, Histogram, Gauge, start_http_server

REQUEST_COUNT = Counter("http_requests_total", "Total HTTP requests",
                        labelnames=["method", "endpoint", "status"])
REQUEST_DURATION = Histogram("http_request_duration_seconds", "Request latency",
                             labelnames=["method", "endpoint"])
IN_PROGRESS = Gauge("http_requests_in_progress", "Requests in progress")

def handle_request(request):
    IN_PROGRESS.inc()
    try:
        with REQUEST_DURATION.labels(method=request.method, endpoint=request.path).time():
            response = process(request)
        REQUEST_COUNT.labels(method=request.method, endpoint=request.path,
                             status=response.status).inc()
        return response
    except Exception:
        REQUEST_COUNT.labels(method=request.method, endpoint=request.path, status="500").inc()
        raise
    finally:
        IN_PROGRESS.dec()

start_http_server(port=8000)  # expose /metrics for Prometheus
```

## Distributed Tracing

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.propagate import inject, extract

# Configure once at startup
resource = Resource.create({"service.name": "order-service", "service.version": "1.2.0"})
provider = TracerProvider(resource=resource)
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
trace.set_tracer_provider(provider)
tracer = trace.get_tracer(__name__)

# Create spans for units of work
def process_order(order):
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order.id)
        span.set_attribute("order.item_count", len(order.items))
        validate_order(order)       # child spans nest automatically
        charge_payment(order)
        span.add_event("payment_charged", {"provider": "stripe"})
        return fulfill(order)

# Propagate context in outgoing HTTP calls
import requests
def call_inventory_service(item_ids):
    with tracer.start_as_current_span("check_inventory") as span:
        headers = {}
        inject(headers)  # injects traceparent/tracestate
        resp = requests.post("http://inventory-service/check",
                             json={"item_ids": item_ids}, headers=headers)
        span.set_attribute("http.status_code", resp.status_code)
        return resp.json()

# Extract context from incoming request
def handle_incoming(request):
    ctx = extract(request.headers)
    with tracer.start_as_current_span("handle_request", context=ctx):
        return process(request)
```

## Alerting

```python
# Alerting rules live in Prometheus config; the service emits SLI-friendly metrics.
# Example Prometheus rule consuming the metrics defined above:
#
#   - alert: OrderServiceHighErrorRate
#     expr: |
#       sum(rate(http_requests_total{status=~"5.."}[5m]))
#       / sum(rate(http_requests_total[5m])) > 0.005
#     for: 5m
#     annotations:
#       runbook_url: "https://runbooks.example.com/order-service/high-error-rate"
```

## Systematic Debugging

```python
import logging, traceback

# Capture full exception context in structured logs
try:
    result = process_order(order)
except Exception:
    logger.exception("order_processing_failed", order_id=order.id)
    raise

# Drop into debugger at a specific point (Python 3.7+)
def debug_failing_transform(data):
    intermediate = step_one(data)
    breakpoint()  # opens pdb; set PYTHONBREAKPOINT=0 to disable in prod
    return step_two(intermediate)

# Per-module log levels for local debugging
logging.basicConfig(level=logging.WARNING)
logging.getLogger("order_service.payment").setLevel(logging.DEBUG)
```

## Dashboards and Operational Readiness

```python
# Health check endpoint for load balancers and dashboards
def health_check():
    checks = {
        "database": check_db_connection(),
        "cache": check_redis_connection(),
    }
    all_healthy = all(checks.values())
    return {"status": "healthy" if all_healthy else "degraded", "checks": checks}

# Verify telemetry pipeline at startup
def verify_observability():
    assert trace.get_tracer_provider() is not None, "Tracer provider not configured"
    logger.info("observability_ready", tracing=True, metrics=True)
```
