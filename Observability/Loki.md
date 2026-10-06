Loki stores logs. Its trick is that it does not index the log text, only a small set of labels. That makes it dramatically cheaper than Elasticsearch, with one trade-off you need to understand.

The core idea

```
  ELASTICSEARCH                  LOKI
  ─────────────                  ────
  indexes every word             indexes only labels
  huge index, often bigger       tiny index
  than the logs themselves       logs stored as compressed chunks

  fast arbitrary text search     finds the stream by label,
                                 then greps the chunks

  expensive                      cheap
```

So Loki is fast when you narrow by labels first, and slow if you ask it to grep everything. "Errors in the checkout service in the last hour" is instant. "This string anywhere, ever" is not what it is for.

Streams and labels

A stream is one unique combination of labels. All logs from that combination go into the same chunk.

```
  {app="checkout", env="prod", level="error"}   <- one stream
  {app="checkout", env="prod", level="info"}    <- a different stream
```

Same cardinality rule as [[Prometheus-Mimir]]: labels must be low cardinality. Never label by user id, request id, or trace id. Those go in the log line itself, where you can still filter on them.

```
  GOOD labels                    BAD labels
  ───────────                    ──────────
  app, env, namespace            user_id
  level, cluster                 request_id
  job, instance                  trace_id, path with ids
```

Getting logs in

Easiest path is to log JSON to stdout and let the collector pick it up.

```rust
/*
1. JSON output means Loki can parse fields at query time
2. trace_id in the line is what links this log to a trace in Tempo
3. never put trace_id in a Loki *label*, only in the line
*/

tracing_subscriber::fmt()
    .json()
    .with_current_span(true)
    .init();

tracing::error!(
    user_id = 91,
    order_id = "ord_88f",
    "payment declined"
);
```

Produces something like:

```json
{"timestamp":"2026-07-21T18:04:11Z","level":"ERROR","fields":{"user_id":91,"order_id":"ord_88f","message":"payment declined"},"trace_id":"a1b2c3"}
```

LogQL

LogQL is PromQL's shape applied to logs. Always starts with a label selector in braces.

```logql
# 1. select the stream (mandatory, this is the indexed part)
{app="checkout", env="prod"}

# 2. filter the line
{app="checkout"} |= "payment declined"      # contains
{app="checkout"} != "healthcheck"           # does not contain
{app="checkout"} |~ "timeout|refused"       # regex match

# 3. parse structured fields out of the line
{app="checkout"} | json | user_id = 91

# 4. now filter on parsed fields
{app="checkout"} | json | level="ERROR" | duration_ms > 500

# 5. pull out just what you want to read
{app="checkout"} | json | line_format "{{.user_id}} -> {{.message}}"
```

Order matters and it is a performance thing: select narrowly, then line-filter (`|=`) to cut volume, then parse (`| json`) last. Parsing is the expensive step, so you want it running on as few lines as possible.

Metrics from logs

You can turn logs into graphs, which is useful when you never added a proper metric.

```logql
# error lines per second
rate({app="checkout"} |= "ERROR" [5m])

# count by level, as a graph
sum by (level) (count_over_time({app="checkout"} | json [5m]))

# p95 of a duration field parsed out of the log line
quantile_over_time(0.95,
  {app="checkout"} | json | unwrap duration_ms [5m]
) by (route)
```

Handy, but a real metric in [[Prometheus-Mimir]] is cheaper if you query it often.

Collecting from Kubernetes

```
// Alloy: discover pods, tail their logs, ship to Loki
discovery.kubernetes "pods" {
  role = "pod"
}

loki.source.kubernetes "pods" {
  targets    = discovery.kubernetes.pods.targets
  forward_to = [loki.write.default.receiver]
}

loki.write "default" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
```

Retention

Logs are the highest-volume signal you have. Set retention before you turn it on, not after the bill arrives.

```yaml
limits_config:
  retention_period: 720h          # 30 days

compactor:
  retention_enabled: true

# keep some streams longer than the default
overrides:
  audit:
    retention_period: 8760h        # 1 year for audit logs
```

Linking logs to traces

This is the payoff for putting `trace_id` in the line. In [[Grafana]] you configure a derived field on the Loki datasource, which turns the id into a clickable link straight into [[Tempo]].

```
  log line in Loki                          trace in Tempo
  ────────────────                          ──────────────
  "payment declined" trace_id=a1b2c3  ────► [ POST /checkout   520ms ]
                                               [ charge card   410ms ]
```

You go from "this errored" to "here is exactly where the time went and what else happened in that request" in one click. That is the whole reason to run these three tools together.

Fed by [[Alloy or Otel Collector]]. Queried in [[Grafana]]. Links across to [[Tempo]].
