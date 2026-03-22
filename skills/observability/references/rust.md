# Rust Observability Reference

Idiomatic Rust patterns for observability. Uses the `tracing` crate for structured logging and spans, the `metrics` crate for metrics, and the OpenTelemetry Rust SDK for distributed tracing export. The `tracing` ecosystem unifies logging and span-based instrumentation into a single API.

---

## Structured Logging

```rust
use tracing::{info, error, debug, instrument, info_span};
use tracing_subscriber::{fmt, EnvFilter, layer::SubscriberExt, util::SubscriberInitExt};

// Configure once at startup; RUST_LOG env var controls verbosity
fn init_logging() {
    tracing_subscriber::registry()
        .with(EnvFilter::from_default_env())
        .with(fmt::layer().json())  // structured JSON output
        .init();
}

// Structured key-value fields — typed, not string-formatted
info!(order_id = %order.id, total = order.total, items = order.items.len(), "order processed");
error!(order_id = %order.id, error = %e, provider = "stripe", "payment failed");

// Spans: context attached to a block of work; child logs inherit span fields
let span = info_span!("handle_request", request_id = %req.id, method = %req.method);
let _guard = span.enter();
info!("request started");  // carries request_id and method automatically

// #[instrument] macro: auto-creates a span for a function
#[instrument(skip(db), fields(user_id = %user_id))]
fn fetch_user(db: &Database, user_id: &str) -> Result<User, DbError> {
    db.query_user(user_id)
}
```

## Metrics and Instrumentation

```rust
use metrics::{counter, gauge, histogram, describe_counter, describe_gauge, describe_histogram};
use metrics_exporter_prometheus::PrometheusBuilder;

fn init_metrics() {
    PrometheusBuilder::new()
        .with_http_listener(([0, 0, 0, 0], 9000))
        .install()
        .expect("failed to install metrics exporter");
    describe_counter!("http_requests_total", "Total HTTP requests");
    describe_histogram!("http_request_duration_seconds", "Request latency");
    describe_gauge!("http_requests_in_progress", "Requests in progress");
}

fn handle_request(req: &Request) -> Response {
    gauge!("http_requests_in_progress").increment(1.0);
    let start = std::time::Instant::now();

    let response = process(req);

    histogram!("http_request_duration_seconds",
        "method" => req.method.clone()).record(start.elapsed().as_secs_f64());
    counter!("http_requests_total",
        "method" => req.method.clone(), "status" => response.status.to_string()).increment(1);
    gauge!("http_requests_in_progress").decrement(1.0);
    response
}
```

## Distributed Tracing

```rust
use opentelemetry::trace::TracerProvider as _;
use opentelemetry_sdk::trace::TracerProvider;
use opentelemetry_otlp::SpanExporter;
use opentelemetry_sdk::Resource;
use opentelemetry::KeyValue;
use tracing_opentelemetry::OpenTelemetryLayer;

fn init_tracing() -> TracerProvider {
    let exporter = SpanExporter::builder().with_tonic().build()
        .expect("failed to create OTLP exporter");
    let provider = TracerProvider::builder()
        .with_batch_exporter(exporter)
        .with_resource(Resource::new(vec![
            KeyValue::new("service.name", "order-service"),
        ]))
        .build();
    let tracer = provider.tracer("order-service");

    tracing_subscriber::registry()
        .with(EnvFilter::from_default_env())
        .with(fmt::layer().json())
        .with(OpenTelemetryLayer::new(tracer))  // exports spans via OTLP
        .init();
    provider
}

// #[instrument] spans are exported as OpenTelemetry spans automatically
#[instrument(fields(order.id = %order.id, order.item_count = order.items.len()))]
fn process_order(order: &Order) -> Result<Receipt, OrderError> {
    validate_order(order)?;
    let receipt = charge_payment(order)?;
    info!(provider = "stripe", "payment charged");  // span event
    Ok(receipt)
}

// Context propagation for outgoing HTTP calls
use opentelemetry::global;
use opentelemetry::propagation::Injector;

struct HeaderInjector<'a>(&'a mut reqwest::header::HeaderMap);
impl<'a> Injector for HeaderInjector<'a> {
    fn set(&mut self, key: &str, value: String) {
        self.0.insert(
            reqwest::header::HeaderName::from_bytes(key.as_bytes()).unwrap(),
            reqwest::header::HeaderValue::from_str(&value).unwrap(),
        );
    }
}

async fn call_inventory(ids: &[String]) -> reqwest::Result<InventoryResponse> {
    let mut headers = reqwest::header::HeaderMap::new();
    global::get_text_map_propagator(|p| p.inject(&mut HeaderInjector(&mut headers)));
    reqwest::Client::new().post("http://inventory-service/check")
        .headers(headers).json(&ids).send().await?.json().await
}
```

## Alerting

```rust
// Alerting rules live in Prometheus config. The service emits SLI-friendly metrics.
fn handle(req: &Request) -> Response {
    counter!("http_requests_total", "type" => "all").increment(1);
    match process(req) {
        Ok(resp) => { counter!("http_requests_total", "type" => "success").increment(1); resp }
        Err(e) => { error!(error = %e, "request failed"); Response::internal_error() }
    }
}
```

## Systematic Debugging

```rust
// RUST_LOG: control verbosity per module
// $ RUST_LOG=warn,order_service::payment=debug cargo run
// $ RUST_BACKTRACE=1 cargo run   # backtrace on panic

fn init_debug_logging() {
    let filter = EnvFilter::new("warn")
        .add_directive("order_service::payment=debug".parse().unwrap());
    tracing_subscriber::registry()
        .with(filter)
        .with(fmt::layer().pretty())  // human-readable colored output
        .init();
}

// Debug-only tracing: use debug!/trace! for internal detail
#[instrument(level = "debug", skip(payload))]
fn transform_payload(payload: &[u8]) -> Result<Transformed, TransformError> {
    debug!(payload_len = payload.len(), "starting transform");
    let result = parse(payload)?;
    debug!(fields = result.field_count(), "transform complete");
    Ok(result)
}
```

## Dashboards and Operational Readiness

```rust
use axum::Json;
use serde_json::{json, Value};

async fn health_check(db: &Database, cache: &Cache) -> Json<Value> {
    let db_ok = db.ping().await.is_ok();
    let cache_ok = cache.ping().await.is_ok();
    Json(json!({"status": if db_ok && cache_ok {"healthy"} else {"degraded"},
                "checks": {"database": db_ok, "cache": cache_ok}}))
}

// Flush pending spans on shutdown
async fn shutdown(provider: TracerProvider) {
    provider.shutdown().expect("failed to shut down tracer provider");
}
```
