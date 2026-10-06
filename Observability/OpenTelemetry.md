OpenTelemetry (OTel) is not a database and not a UI. It is the agreed-on standard for how your code emits telemetry, so you can swap backends later without rewriting your app.

Before OTel you wrote code against a vendor's SDK. Change vendor, rewrite everything. OTel makes the app speak one language, and the backend becomes a config line.

The three signals

```
  METRICS   a number over time          "requests/sec is 240"
            cheap, aggregated           answers: is something wrong?

  LOGS      a timestamped line          "user 91 checkout failed: card declined"
            detailed, expensive         answers: what exactly happened?

  TRACES    the path of one request     "this request spent 400ms in the DB"
            shows causality             answers: where is the time going?
```

You want all three. Metrics tell you something broke, traces tell you where, logs tell you why.

What a trace actually is

A trace is one request's journey through your system. Each step is a span.

```
  trace_id: a1b2c3
  ──────────────────────────────────────────────────────────
  [ POST /checkout                                  520ms ]   root span
     [ auth check          12ms ]
     [ SELECT user          8ms ]
     [ charge card                        410ms ]             <- the problem
        [ POST stripe.com               405ms ]
     [ INSERT order        30ms ]
     [ publish to queue     9ms ]
  ──────────────────────────────────────────────────────────
```

Instantly obvious that the card charge dominates. A log line could never show you that shape.

Spans nest because each one carries its parent's id. A span is just: name, start time, end time, parent id, and a bag of key-value attributes.

Context propagation

This is the part people get wrong. For a trace to survive across services, the trace id must ride along with the network call, normally in the `traceparent` HTTP header.

```
  service A                    service B
  ─────────                    ─────────
  span (trace a1b2c3)
      │
      │  HTTP POST /pay
      │  traceparent: 00-a1b2c3-<span_id>-01
      ▼
                               span (trace a1b2c3, parent = span_id)
```

Same trace id on both sides, so the backend can stitch them into one timeline. Forget the header and you get two unrelated traces and a very confusing afternoon.

Auto vs manual instrumentation

Auto-instrumentation patches known libraries (your HTTP server, DB driver, HTTP client) and gives you spans for free. Start here, it covers most of what you need.

Manual instrumentation is you adding spans around your own business logic.

Minimal setup in Rust

```rust
/*
1. a tracer provider decides where spans go and how they are batched
2. the OTLP exporter ships them over gRPC to a collector
3. the tracing_subscriber layer bridges Rust's tracing crate to OTel
4. after this, #[instrument] on any fn creates a span
*/

use opentelemetry_otlp::WithExportConfig;
use tracing_subscriber::layer::SubscriberExt;

fn init_tracing() -> anyhow::Result<()> {
    let tracer = opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_exporter(
            opentelemetry_otlp::new_exporter()
                .tonic()
                .with_endpoint("http://localhost:4317"),   // the collector
        )
        .install_batch(opentelemetry_sdk::runtime::Tokio)?;

    let subscriber = tracing_subscriber::registry()
        .with(tracing_subscriber::EnvFilter::new("info"))
        .with(tracing_opentelemetry::layer().with_tracer(tracer));

    tracing::subscriber::set_global_default(subscriber)?;
    Ok(())
}
```

Then annotate your handlers:

```rust
/*
1. #[instrument] wraps the fn body in a span named after the fn
2. skip(db) keeps the connection pool out of the attributes
3. fields you add become searchable attributes on the span
*/

#[tracing::instrument(skip(db), fields(user_id = %id))]
async fn get_user(db: &Pool, id: i64) -> Result<User> {
    tracing::info!("loading user");     // this log attaches to the span
    let user = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", id)
        .fetch_one(db)
        .await?;
    Ok(user)
}
```

OTLP, the wire protocol

OTLP is how telemetry travels. Two ports you will type constantly:

```
  4317   OTLP over gRPC     default, faster
  4318   OTLP over HTTP     easier through proxies and from browsers
```

Everything downstream speaks it: [[Alloy or Otel Collector]], [[Tempo]], and increasingly [[Prometheus-Mimir]] and [[Loki]].

Attributes worth setting

`service.name` is the one that matters most. Without it everything shows up as `unknown_service` and you cannot tell your services apart.

```bash
export OTEL_SERVICE_NAME=checkout-api
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_RESOURCE_ATTRIBUTES=deployment.environment=prod,service.version=1.4.2
```

Most SDKs read these env vars with no code changes at all.

Sampling

At scale you cannot store every trace. Sampling keeps a fraction.

```
  head sampling    decide at the start, e.g. keep 10%
                   cheap, but you might drop the one broken request

  tail sampling    buffer the whole trace, then decide
                   keeps all errors and slow requests, costs memory
                   done in the collector, not the app
```

Start at 100% in dev, then move to tail sampling in the collector when volume hurts.

Where this goes next

Your app exports OTLP to [[Alloy or Otel Collector]], which fans the signals out to [[Prometheus-Mimir]], [[Loki]] and [[Tempo]], and you look at all of it in [[Grafana]].
