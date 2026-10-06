Prometheus stores metrics: numbers measured over time. It is the default answer for "is the system healthy right now".

Mimir is Prometheus for when one server is not enough. Same query language, same data model, but storage moves to object storage (S3) and scales sideways.

Pull, not push

Almost every other tool pushes data out. Prometheus goes and fetches it.

```
                 every 15s
  Prometheus ──────────────► GET http://your-app:8080/metrics
             ◄──────────────
                 text response
```

Your app just exposes a plain HTTP endpoint that prints its current numbers. Prometheus scrapes it on a timer.

This sounds backwards and is actually great: Prometheus knows immediately when a target stops responding (that becomes the `up` metric), and your app never needs to know Prometheus exists.

The exception is short-lived jobs that die before a scrape. Those push to a Pushgateway, or you use OTLP push via [[Alloy or Otel Collector]].

What a metric looks like

```
  http_requests_total{method="POST", route="/checkout", status="500"}  1027
  └────────┬───────┘ └──────────────────┬──────────────────────────┘  └─┬┘
     metric name                      labels                         value
```

The labels are the whole point. Same metric name, different label combinations, each one is its own time series you can filter and group by.

The four metric types

```
  COUNTER     only goes up, resets to 0 on restart
              requests served, errors, bytes sent
              you almost always wrap it in rate()

  GAUGE       goes up and down
              memory in use, queue depth, temperature
              read it directly

  HISTOGRAM   counts observations into buckets
              request duration, response size
              lets you compute percentiles server-side

  SUMMARY     like histogram but percentiles computed in the app
              can't be aggregated across instances, prefer histogram
```

Exposing metrics

```rust
use actix_web::{HttpRequest, Responder};

/*
1. a registry holds every metric in the process
2. counters and histograms get registered once at startup
3. the /metrics handler serialises the whole registry as text
*/

use prometheus::{register_counter_vec, register_histogram_vec};

lazy_static! {
    static ref REQUESTS: CounterVec = register_counter_vec!(
        "http_requests_total",
        "Total HTTP requests",
        &["method", "route", "status"]      // label names
    ).unwrap();

    static ref DURATION: HistogramVec = register_histogram_vec!(
        "http_request_duration_seconds",
        "Request duration",
        &["route"],
        vec![0.005, 0.01, 0.05, 0.1, 0.5, 1.0, 5.0]   // bucket edges
    ).unwrap();
}

async fn handler(req: HttpRequest) -> impl Responder {
    let timer = DURATION.with_label_values(&["/checkout"]).start_timer();
    let res = do_work().await;
    timer.observe_duration();

    REQUESTS.with_label_values(&["POST", "/checkout", "200"]).inc();
    res
}
```

Scrape config

```yaml
global:
  scrape_interval: 15s          # how often to fetch

scrape_configs:
  - job_name: checkout-api
    static_configs:
      - targets: ['checkout-api:8080']

  # in Kubernetes you discover targets instead of listing them
  - job_name: k8s-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # only scrape pods that opted in with an annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

PromQL, the bits you actually use

Counters are useless raw. `rate()` converts them to per-second.

```promql
# requests per second over the last 5 minutes
rate(http_requests_total[5m])

# error rate as a percentage, summed across instances
sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
sum(rate(http_requests_total[5m])) * 100

# p95 latency from a histogram
histogram_quantile(0.95,
  sum by (le, route) (rate(http_request_duration_seconds_bucket[5m]))
)

# memory per pod, top 5
topk(5, container_memory_working_set_bytes)

# is anything down?
up == 0
```

`[5m]` is a range selector, it means "look back 5 minutes". `rate` needs one. `sum by (le, ...)` before `histogram_quantile` is mandatory, and forgetting `le` is the single most common PromQL bug.

Cardinality, the thing that will bite you

Every unique combination of labels is a separate stored series. Put something unbounded in a label and you get an explosion.

```
  FINE                                     DISASTER
  ────                                     ────────
  route="/checkout"      ~20 values        user_id="91823"     millions
  status="500"           ~10 values        request_id="a1b2"   infinite
  method="POST"          ~6 values         email="..."         infinite

  20 x 10 x 6 = 1200 series                OOM, then a bad evening
```

Rule of thumb: a label value must come from a small, fixed set. High-cardinality detail belongs on a trace ([[Tempo]]) or a log line ([[Loki]]), not a metric.

When you outgrow Prometheus

A single Prometheus is limited by one machine's disk and RAM, and it has no real long-term storage or HA.

```
  Prometheus            Mimir
  ──────────            ─────
  local disk            object storage (S3 / GCS)
  ~weeks of data        years
  one process           microservices, replicated
  no tenancy            multi-tenant out of the box
  simple                needs real operational effort
```

Mimir speaks the same PromQL and accepts Prometheus remote-write, so migrating is mostly pointing your writes at it. Do not start with Mimir. Start with Prometheus and move when it actually hurts.

Alerting

```yaml
groups:
  - name: api
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
            / sum(rate(http_requests_total[5m])) > 0.05
        for: 10m                    # must stay true this long, kills flapping
        labels:
          severity: page
        annotations:
          summary: "5xx rate above 5% for 10 minutes"
```

`for:` is the important field. Without it you get paged for every one-second blip.

Queried through [[Grafana]]. Fed by [[Alloy or Otel Collector]] or scraped directly.
