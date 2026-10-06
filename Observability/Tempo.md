Tempo stores traces. Like [[Loki]] it keeps the index tiny: originally you could only look a trace up by its id, everything else came from metrics and logs pointing at it.

Modern Tempo also has TraceQL, so you can search by attributes, but the design instinct is the same. Cheap object storage, minimal index.

What it is for

Metrics tell you the checkout endpoint got slow. Traces tell you which of the nine things it does got slow.

```
  [ POST /checkout                                  520ms ]
     [ auth check          12ms ]
     [ SELECT user          8ms ]
     [ charge card                        410ms ]     <- here
        [ POST stripe.com               405ms ]       <- actually here
     [ INSERT order        30ms ]
```

Without this you are guessing, or adding log lines and redeploying until you find it.

Running it

Tempo is a single binary and genuinely easy to start with.

```yaml
server:
  http_listen_port: 3200

distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317

# generate RED metrics from spans automatically
metrics_generator:
  registry:
    external_labels:
      source: tempo
  storage:
    path: /var/tempo/generator/wal
    remote_write:
      - url: http://mimir:9009/api/v1/push

storage:
  trace:
    backend: s3
    s3:
      bucket: tempo-traces
      endpoint: s3.eu-west-2.amazonaws.com

overrides:
  defaults:
    metrics_generator:
      processors: [service-graphs, span-metrics]
```

That `metrics_generator` block is worth turning on. It derives request rate, error rate and duration for every service from the spans themselves and writes them into [[Prometheus-Mimir]]. You get RED metrics without instrumenting a single counter.

TraceQL

Same braces-first shape as LogQL. You are selecting spans, then filtering.

```traceql
# any trace with a span from this service
{ resource.service.name = "checkout-api" }

# slow ones only
{ resource.service.name = "checkout-api" && duration > 1s }

# errors
{ status = error }

# a specific user's failed requests
{ span.user_id = 91 && status = error }

# traces that touched the database AND took over 2s overall
{ span.db.system = "postgresql" } && { duration > 2s }

# aggregate: p95 duration grouped by route
{ name = "GET /api/*" } | quantile_over_time(duration, 0.95) by (span.http.route)
```

Attributes are namespaced. `resource.*` describes the service emitting the span, `span.*` is on the individual span. Mixing them up returns nothing and no error, which is confusing the first time.

Service graphs

From span parent/child relationships across services, Tempo can draw what actually calls what.

```
   ┌──────────┐      ┌──────────┐      ┌──────────┐
   │ frontend │─────►│ checkout │─────►│ payments │
   └──────────┘      └────┬─────┘      └────┬─────┘
                          │                 │
                          ▼                 ▼
                     ┌─────────┐       ┌────────┐
                     │ postgres│       │ stripe │
                     └─────────┘       └────────┘
```

Useful because it is derived from real traffic, not from a diagram someone drew two years ago and never updated.

Exemplars, the good trick

An exemplar is a trace id attached to a metric data point. You are looking at a p99 latency graph, you see a spike, you click the dot, and you land in the exact trace that caused it.

```
   p99 latency
       │              ●  <- exemplar, click it
       │             ╱ ╲
       │  ──────────    ────────
       └───────────────────────────► time
                    │
                    ▼
       [ POST /checkout          3.2s ]
          [ charge card         3.1s ]
```

This requires the histogram to be recorded with exemplars enabled and [[Grafana]] wired to link Mimir to Tempo. It is the single best debugging shortcut in the whole stack.

The three-way link

Set this up once and you can move between all three signals freely:

```
  METRIC spike        ──exemplar──►  TRACE          ──span links──►  LOGS
  (Mimir)                            (Tempo)                         (Loki)
  "p99 jumped"                       "stripe call                    "card network
                                      took 3.1s"                      timeout"
```

In Grafana this is the Loki datasource's derived field (log to trace) plus Tempo's "logs for this span" config (trace back to logs), keyed on `trace_id`.

Sampling reminder

Traces are big. See the tail sampling section in [[Alloy or Otel Collector]] for keeping all the errors and slow requests while dropping most of the healthy ones.

Fed by [[Alloy or Otel Collector]] and [[OpenTelemetry]]. Viewed in [[Grafana]].
