Clickhouse is a columnar database built for analytics. In an observability stack it is the alternative to running [[Loki]] plus [[Tempo]] plus [[Prometheus-Mimir]]: one database, plain SQL, all three signals in tables.

Why columnar is fast

A row store keeps each row together. A column store keeps each column together.

```
  ROW STORE (Postgres)              COLUMN STORE (Clickhouse)
  ────────────────────              ─────────────────────────
  [id|ts|level|msg|dur]             [id,id,id,id,id,...]
  [id|ts|level|msg|dur]             [ts,ts,ts,ts,ts,...]
  [id|ts|level|msg|dur]             [level,level,level,...]
                                    [msg,msg,msg,msg,...]
                                    [dur,dur,dur,dur,...]

  SELECT avg(dur) reads             SELECT avg(dur) reads
  every column of every row         only the dur column
```

Two consequences. You skip the columns you did not ask for, and each column holds one data type with similar values, so compression is enormous. Log levels compress to almost nothing because it is the same handful of strings over and over.

Trade-off: brilliant at scanning millions of rows to aggregate, bad at updating or deleting single rows. That matches telemetry perfectly, since it is append-only and you always query it in aggregate.

MergeTree

`MergeTree` is the engine you will use. The important part is `ORDER BY`, which decides physical sort order on disk and therefore what queries are fast.

```sql
CREATE TABLE logs (
    timestamp    DateTime64(3),
    service      LowCardinality(String),   -- dictionary encoded, tiny
    level        LowCardinality(String),
    trace_id     String,
    message      String,
    attributes   Map(String, String)       -- schema-less extras
)
ENGINE = MergeTree
ORDER BY (service, level, timestamp)       -- decides query speed
PARTITION BY toDate(timestamp)             -- lets you drop whole days cheaply
TTL timestamp + INTERVAL 30 DAY;           -- automatic retention
```

Three things worth copying:

`LowCardinality(String)` for anything with a small fixed set of values. It stores an integer per row plus one dictionary. Big win on service names and log levels.

`ORDER BY` should go from coarse to fine, ending in `timestamp`. Queries that filter on a prefix of that tuple can skip most of the data. Filter on something not in the ORDER BY and you full-scan.

`TTL` gives you retention for free, and `PARTITION BY` day means dropping old data is a metadata operation rather than a delete.

Querying

It is just SQL, which is the main selling point over learning PromQL and LogQL and TraceQL.

```sql
-- error rate per service, 5 minute buckets
SELECT
    toStartOfFiveMinute(timestamp) AS t,
    service,
    countIf(level = 'ERROR') / count() AS error_rate
FROM logs
WHERE timestamp > now() - INTERVAL 1 HOUR
GROUP BY t, service
ORDER BY t;

-- p95 latency, no histogram buckets needed
SELECT
    service,
    quantile(0.95)(duration_ms) AS p95,
    count() AS n
FROM spans
WHERE timestamp > now() - INTERVAL 1 HOUR
GROUP BY service
ORDER BY p95 DESC;

-- reconstruct one trace
SELECT span_id, parent_span_id, name, duration_ms
FROM spans
WHERE trace_id = 'a1b2c3'
ORDER BY timestamp;

-- pull a value out of the attributes map
SELECT attributes['user_id'] AS user, count()
FROM logs
WHERE level = 'ERROR'
GROUP BY user
ORDER BY count() DESC
LIMIT 10;
```

Note `quantile(0.95)(duration_ms)` computed from raw values. No pre-defined buckets, no `histogram_quantile`, and you can group by anything after the fact. That flexibility is the thing you lose with metrics.

Materialised views for rollups

Raw data gets expensive to scan. A materialised view aggregates on insert, so dashboards read a tiny table.

```sql
/*
1. the target table stores per-minute aggregates
2. AggregatingMergeTree merges partial states as data arrives
3. the view fires on every insert into spans
*/

CREATE TABLE spans_1m (
    minute   DateTime,
    service  LowCardinality(String),
    count    AggregateFunction(count),
    p95      AggregateFunction(quantile(0.95), Float64)
)
ENGINE = AggregatingMergeTree
ORDER BY (service, minute);

CREATE MATERIALIZED VIEW spans_1m_mv TO spans_1m AS
SELECT
    toStartOfMinute(timestamp) AS minute,
    service,
    countState()                    AS count,
    quantileState(0.95)(duration_ms) AS p95
FROM spans
GROUP BY minute, service;

-- read it back with -Merge
SELECT service, countMerge(count), quantileMerge(0.95)(p95)
FROM spans_1m
WHERE minute > now() - INTERVAL 6 HOUR
GROUP BY service;
```

Writing into it

Clickhouse hates lots of tiny inserts. Each one creates a part that has to be merged later. Batch aggressively, which the collector already does for you.

```yaml
exporters:
  clickhouse:
    endpoint: tcp://clickhouse:9000
    database: otel
    ttl: 720h

processors:
  batch:
    timeout: 5s
    send_batch_size: 10000     # much bigger than for other backends
```

When to pick it over the Grafana stack

```
  GRAFANA STACK                  CLICKHOUSE
  ─────────────                  ──────────
  purpose-built per signal       one engine for everything
  3 query languages              SQL
  3 systems to operate           1 system to operate
  great out-of-box dashboards    you build more yourself
  Loki cheap for raw logs        better for high-cardinality
                                 analytical questions

  easier to start                easier to ask arbitrary questions
```

If your questions are mostly "show me the standard dashboards and let me drill down", the Grafana stack wins. If you keep wanting to join telemetry against business data and ask questions nobody anticipated, Clickhouse wins. It is also what several commercial observability vendors run underneath.

Either way you can point [[Grafana]] at it with the Clickhouse datasource plugin, so the front end does not change.

Fed by [[Alloy or Otel Collector]]. Visualised in [[Grafana]]. Compare with [[Loki]], [[Tempo]] and [[Prometheus-Mimir]].
