# C++ Observability Reference

Idiomatic C++ patterns for observability. Uses `spdlog` for structured logging, the Prometheus C++ client for metrics, and the OpenTelemetry C++ SDK for distributed tracing. Debugging leverages GDB and compiler sanitizers.

---

## Structured Logging

```cpp
#include <spdlog/spdlog.h>
#include <spdlog/sinks/stdout_color_sinks.h>

auto logger = spdlog::stdout_color_mt("order-service");

// Structured key-value pairs embedded in the message via fmt
logger->info("order_processed order_id={} total={:.2f} items={}", order.id, order.total, order.items.size());
logger->error("payment_failed order_id={} error={}", order.id, exc.what());

// JSON output via custom pattern
logger->set_pattern(R"({"time":"%Y-%m-%dT%H:%M:%S.%eZ","level":"%l","msg":"%v"})");

// Per-request context: clone logger with bound pattern
auto req_log = logger->clone("order-service");
req_log->set_pattern("[%Y-%m-%d %H:%M:%S] [%l] [rid=" + request_id + "] %v");
req_log->info("request_started method=POST path=/orders");
```

## Metrics and Instrumentation

```cpp
#include <prometheus/counter.h>
#include <prometheus/histogram.h>
#include <prometheus/gauge.h>
#include <prometheus/exposer.h>
#include <prometheus/registry.h>

auto registry = std::make_shared<prometheus::Registry>();
prometheus::Exposer exposer{"0.0.0.0:8080"};
exposer.RegisterCollectable(registry);

auto& request_counter = prometheus::BuildCounter()
    .Name("http_requests_total").Help("Total HTTP requests").Register(*registry);
auto& request_duration = prometheus::BuildHistogram()
    .Name("http_request_duration_seconds").Help("Request latency").Register(*registry);
auto& in_progress = prometheus::BuildGauge()
    .Name("http_requests_in_progress").Help("Requests in progress").Register(*registry);

void handle_request(const Request& req) {
    auto& counter = request_counter.Add({{"method", req.method}, {"endpoint", req.path}});
    auto& duration = request_duration.Add(
        {{"method", req.method}},
        prometheus::Histogram::BucketBoundaries{0.01, 0.05, 0.1, 0.25, 0.5, 1.0});
    auto& gauge = in_progress.Add({});

    gauge.Increment();
    auto start = std::chrono::steady_clock::now();
    auto response = process(req);
    duration.Observe(std::chrono::duration<double>(std::chrono::steady_clock::now() - start).count());
    counter.Increment();
    gauge.Decrement();
}
```

## Distributed Tracing

```cpp
#include <opentelemetry/trace/provider.h>
#include <opentelemetry/sdk/trace/tracer_provider.h>
#include <opentelemetry/sdk/trace/simple_processor.h>
#include <opentelemetry/exporters/otlp/otlp_grpc_exporter_factory.h>
#include <opentelemetry/trace/propagation/http_trace_context.h>
#include <opentelemetry/context/propagation/global_propagator.h>

namespace trace_api = opentelemetry::trace;
namespace trace_sdk = opentelemetry::sdk::trace;
namespace otlp = opentelemetry::exporter::otlp;

void init_tracing() {
    auto exporter  = otlp::OtlpGrpcExporterFactory::Create();
    auto processor = trace_sdk::SimpleSpanProcessorFactory::Create(std::move(exporter));
    auto provider  = trace_sdk::TracerProviderFactory::Create(std::move(processor));
    trace_api::Provider::SetTracerProvider(std::move(provider));
    opentelemetry::context::propagation::GlobalTextMapPropagator::SetGlobalPropagator(
        std::make_shared<trace_api::propagation::HttpTraceContext>());
}

void process_order(const Order& order) {
    auto tracer = trace_api::Provider::GetTracerProvider()->GetTracer("order-service");
    auto span = tracer->StartSpan("process_order");
    auto scope = tracer->WithActiveSpan(span);
    span->SetAttribute("order.id", order.id);
    span->SetAttribute("order.item_count", static_cast<int64_t>(order.items.size()));
    validate_order(order);
    charge_payment(order);
    span->AddEvent("payment_charged", {{"provider", "stripe"}});
    span->SetStatus(trace_api::StatusCode::kOk);
    span->End();
}
```

## Alerting

```cpp
// Alerting rules live in Prometheus config. The service emits SLI-friendly metrics.
// Expose separate total/success counters so Prometheus computes the success ratio.
auto& total   = request_counter.Add({{"type", "total"}});
auto& success = request_counter.Add({{"type", "success"}});

void handle(const Request& req) {
    total.Increment();
    auto resp = process(req);   // on failure, total incremented but not success
    success.Increment();
}
```

## Systematic Debugging

```cpp
// GDB: attach to running process or inspect core dump
// $ gdb ./order-service core.12345
// (gdb) bt          # backtrace
// (gdb) frame 3     # inspect frame
// (gdb) info locals  # local variables

// Compile with sanitizers for development builds
// $ g++ -g -fsanitize=address,undefined -o order-service main.cpp
// AddressSanitizer: buffer overflow, use-after-free, leaks
// ThreadSanitizer:  data races (use -fsanitize=thread)

// Per-logger debug levels without recompilation
spdlog::get("payment")->set_level(spdlog::level::debug);
// everything else stays at info

// Debug assertions: active only in debug builds
#include <cassert>
void transfer(Account& from, Account& to, double amount) {
    assert(amount > 0 && "transfer amount must be positive");
    // ... transfer logic ...
    assert(from.balance() >= 0 && "postcondition: balance non-negative");
}
```

## Dashboards and Operational Readiness

```cpp
#include <nlohmann/json.hpp>

nlohmann::json health_check() {
    bool db_ok = check_db_connection();
    bool cache_ok = check_redis_connection();
    return {{"status", (db_ok && cache_ok) ? "healthy" : "degraded"},
            {"checks", {{"database", db_ok}, {"cache", cache_ok}}}};
}

void verify_observability() {
    auto provider = trace_api::Provider::GetTracerProvider();
    if (!provider) { spdlog::critical("Tracer provider not configured"); std::abort(); }
    spdlog::info("observability_ready tracing=true metrics=true");
}
```
