A collector is a middleman between your apps and your storage. Your app sends telemetry to it, it cleans things up, and it forwards them on.

You do not strictly need one. Your app could write straight to [[Tempo]] and [[Prometheus-Mimir]]. You will want one anyway.

Why bother

```
  WITHOUT a collector              WITH a collector
  ──────────────────               ────────────────
  app ──► Tempo                    app ──┐
  app ──► Mimir                    app ──┼─► collector ─┬─► Tempo
  app ──► Loki                     app ──┘              ├─► Mimir
                                                        └─► Loki

  every app knows every            apps know one address
  backend address                  change backends = change one config
  redeploy to change one           retries and buffering happen here
  no buffering if a                strip secrets before they leave
  backend goes down                the network
```

The real win is that the app stops caring where data goes.

Two options that do the same job

**OpenTelemetry Collector** is the vendor-neutral upstream project. YAML config, works with anything.

**Grafana Alloy** is Grafana's distribution, previously called Grafana Agent. Its own config language, components reference each other. Better if you are all-in on the Grafana stack.

Pick either. Alloy is nicer for Grafana-heavy setups, the OTel Collector is more portable.

The pipeline model

Both tools are the same three-stage idea:

```
  RECEIVERS ──► PROCESSORS ──► EXPORTERS
  how data      what you do     where it
  gets in       to it           goes out

  OTLP          batch           Tempo
  Prometheus    filter          Mimir
  scraping      redact          Loki
  file logs     sample          Clickhouse
```

OTel Collector config

```yaml
# 1. receivers: open the doors
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

# 2. processors: run in the order you list them in the pipeline
processors:
  # batch is not optional in practice, it cuts network calls enormously
  batch:
    timeout: 5s
    send_batch_size: 1024

  # protect the collector from OOMing under a traffic spike
  memory_limiter:
    check_interval: 1s
    limit_percentage: 80
    spike_limit_percentage: 20

  # never let card numbers reach storage
  attributes/redact:
    actions:
      - key: credit_card
        action: delete
      - key: http.request.header.authorization
        action: delete

# 3. exporters: where it lands
exporters:
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true
  prometheusremotewrite:
    endpoint: http://mimir:9009/api/v1/push
  loki:
    endpoint: http://loki:3100/loki/api/v1/push

# 4. nothing runs until it is wired into a pipeline
service:
  pipelines:
    traces:
      receivers:  [otlp]
      processors: [memory_limiter, attributes/redact, batch]
      exporters:  [otlp/tempo]
    metrics:
      receivers:  [otlp]
      processors: [memory_limiter, batch]
      exporters:  [prometheusremotewrite]
    logs:
      receivers:  [otlp]
      processors: [memory_limiter, batch]
      exporters:  [loki]
```

The mistake everyone makes once: defining a receiver or processor and forgetting to add it to a pipeline. It is silently ignored.

Same thing in Alloy

Alloy config is components wired by referencing each other's exports. Reads backwards at first, but the dependency graph is explicit.

```
// 1. receive OTLP from apps
otelcol.receiver.otlp "default" {
  grpc { endpoint = "0.0.0.0:4317" }
  http { endpoint = "0.0.0.0:4318" }

  output {
    traces  = [otelcol.processor.batch.default.input]
    metrics = [otelcol.processor.batch.default.input]
  }
}

// 2. batch, then hand to the exporters
otelcol.processor.batch "default" {
  output {
    traces  = [otelcol.exporter.otlp.tempo.input]
    metrics = [otelcol.exporter.prometheus.mimir.input]
  }
}

// 3. out to storage
otelcol.exporter.otlp "tempo" {
  client {
    endpoint = "tempo:4317"
    tls { insecure = true }
  }
}
```

Tail sampling

The collector is the only place tail sampling can happen, because only it sees the whole trace. Keep every error and every slow request, throw away most of the boring ones.

```yaml
processors:
  tail_sampling:
    decision_wait: 10s          # buffer this long before deciding
    policies:
      - name: keep-errors
        type: status_code
        status_code: { status_codes: [ERROR] }

      - name: keep-slow
        type: latency
        latency: { threshold_ms: 1000 }

      - name: sample-the-rest
        type: probabilistic
        probabilistic: { sampling_percentage: 5 }
```

Usually a 90%+ cost cut with no loss of the traces you actually open.

Where to run it

```
  AGENT MODE      one per host or as a sidecar, close to the app
                  short network hop, collects host metrics too

  GATEWAY MODE    a central cluster of collectors
                  agents forward here, this is where tail sampling
                  and egress control live

  most real setups run both:  agent ──► gateway ──► storage
```

Checking it works

The collector exposes metrics about itself. If data is vanishing, look here first.

```bash
# is it dropping anything on export?
curl -s localhost:8888/metrics | grep -E 'otelcol_exporter_send_failed|otelcol_processor_dropped'

# is it receiving anything at all?
curl -s localhost:8888/metrics | grep otelcol_receiver_accepted
```

`send_failed` climbing means the backend is unreachable or rejecting. `refused` on the receiver side usually means memory_limiter is pushing back.

Fed by [[OpenTelemetry]]. Feeds [[Prometheus-Mimir]], [[Loki]], [[Tempo]] and optionally [[Clickhouse]].
