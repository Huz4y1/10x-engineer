Grafana is the front end. It stores no telemetry of its own, it queries the systems that do and draws the result.

That separation is the point: one place to look, whatever is underneath.

```
   ┌───────────────── Grafana ─────────────────┐
   │  dashboards   explore   alerts            │
   └───┬────────┬────────┬────────┬────────────┘
       │        │        │        │
   ┌───▼──┐ ┌───▼──┐ ┌───▼──┐ ┌───▼──────┐
   │Mimir │ │ Loki │ │Tempo │ │Clickhouse│
   │metric│ │ logs │ │traces│ │   all    │
   └──────┘ └──────┘ └──────┘ └──────────┘
```

Datasources

A datasource is a connection to one backend. Define them as code so they are reproducible.

```yaml
# provisioning/datasources/all.yaml
apiVersion: 1
datasources:
  - name: Mimir
    type: prometheus
    uid: mimir                       # uid is what other configs reference
    url: http://mimir:9009/prometheus
    jsonData:
      exemplarTraceIdDestinations:
        - name: trace_id
          datasourceUid: tempo       # makes exemplars clickable

  - name: Loki
    type: loki
    uid: loki
    url: http://loki:3100
    jsonData:
      derivedFields:
        # find trace_id in a log line, turn it into a link to Tempo
        - name: TraceID
          matcherRegex: '"trace_id":"(\w+)"'
          url: '$${__value.raw}'
          datasourceUid: tempo

  - name: Tempo
    type: tempo
    uid: tempo
    url: http://tempo:3200
    jsonData:
      tracesToLogsV2:
        datasourceUid: loki
        filterByTraceID: true        # jump from a span to its logs
      serviceMap:
        datasourceUid: mimir
```

Those three cross-references are what turn separate tools into one workflow. Spend the ten minutes to set them up.

Explore vs dashboards

Explore is the scratchpad. One query, iterate fast, no saving. This is where you actually debug.

Dashboards are the saved view for things you check repeatedly. Build them after you know what matters, not before. A dashboard of 40 panels nobody reads is worse than none.

Panels worth knowing

```
  TIME SERIES    the default, anything over time
  STAT           one big number, current error rate or uptime
  GAUGE          a number against a threshold, disk usage
  TABLE          top-N lists, slowest endpoints
  HEATMAP        distribution over time, latency buckets
  LOGS           raw log lines from Loki
  TRACE          a flame graph from Tempo
  BAR GAUGE      comparing values across services
```

Variables

Variables turn one dashboard into many. Define `$service` once and every panel follows the dropdown.

```
  Type: Query
  Datasource: Mimir
  Query: label_values(http_requests_total, service)
  Multi-value: yes
  Include All: yes
```

Then use it in queries, with `=~` and the regex format so "All" works:

```promql
sum by (route) (
  rate(http_requests_total{service=~"$service"}[5m])
)
```

Chained variables are the nice bit: make `$pod` query `label_values(kube_pod_info{namespace="$namespace"}, pod)` and picking a namespace filters the pod list automatically.

A dashboard that is actually useful

The RED method: Rate, Errors, Duration. Three rows and you understand a service.

```promql
# Rate
sum(rate(http_requests_total{service=~"$service"}[5m]))

# Errors, as a percentage
sum(rate(http_requests_total{service=~"$service", status=~"5.."}[5m]))
  / sum(rate(http_requests_total{service=~"$service"}[5m])) * 100

# Duration, p50 / p95 / p99 on one panel
histogram_quantile(0.95,
  sum by (le) (rate(http_request_duration_seconds_bucket{service=~"$service"}[5m]))
)
```

For infrastructure the equivalent is USE: Utilisation, Saturation, Errors.

Alerting

Grafana's unified alerting can query any datasource, not just metrics, so you can alert on a LogQL query too.

```
  1. QUERY       the data, e.g. error rate over 5m
  2. REDUCE      collapse the series to one number (last, mean, max)
  3. THRESHOLD   is that number above the line
  4. FOR         must stay true this long before firing
  5. ROUTE       contact point, based on labels
```

Two rules that save you from being ignored: always set `for:` so a single blip does not page anyone, and alert on symptoms your users feel (error rate, latency) rather than causes (CPU at 90% is fine if nothing is slow).

Dashboards as code

Clicking dashboards together is fine until you lose them. Export to JSON and commit it.

```yaml
# provisioning/dashboards/all.yaml
apiVersion: 1
providers:
  - name: default
    folder: Services
    type: file
    options:
      path: /etc/grafana/dashboards
    allowUiUpdates: false     # UI edits can't drift from git
```

Also worth knowing: grafana.com has thousands of prebuilt dashboards you import by ID. Node Exporter Full (1860) and Kubernetes cluster monitoring are the usual starting points, and reading their JSON is a good way to learn how the panels are built.

The workflow this all enables

```
  alert fires          "error rate above 5%"
      │
      ▼
  dashboard            which service, when did it start
      │
      ▼
  exemplar / trace     which span is slow          -> [[Tempo]]
      │
      ▼
  logs for that span   the actual error message    -> [[Loki]]
```

Four clicks from "something is wrong" to the exact stack trace. That is what the whole stack is for.

Queries [[Prometheus-Mimir]], [[Loki]], [[Tempo]] and [[Clickhouse]].
