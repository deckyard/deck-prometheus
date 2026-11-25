<!-- SPDX-License-Identifier: LGPL-3.0-only -->
<!-- SPDX-FileCopyrightText: Copyright (C) 2026 mplx <mplx@jennifer-lang.dev> -->

# prometheus cheatsheet

Every export of `@deckyard/prometheus`. The guide is [the introduction](index.md).

```jennifer
import "@deckyard/prometheus/" as prometheus;
```

## Functions

| Function | Returns | Does |
|---|---|---|
| `prometheus.counter(name as string, help as string)` | `Metric` | Build an empty counter metric |
| `prometheus.gauge(name as string, help as string)` | `Metric` | Build an empty gauge metric |
| `prometheus.histogram(name as string, help as string, buckets as list of float)` | `Metric` | Build an empty histogram metric |
| `prometheus.summary(name as string, help as string, quantiles as list of float)` | `Metric` | Build an empty summary metric |
| `prometheus.observe(metric as Metric, labels as map of string to string, value as float)` | `Metric` | Record an observation for a label set, returning a new Metric (value-semantic) |
| `prometheus.observeAt(metric as Metric, labels as map of string to string, value as float, timestampMs as int)` | `Metric` | Record an observation carrying an explicit millisecond timestamp, returning a new Metric (value-semantic) |
| `prometheus.render(metrics as list of Metric)` | `string` | Render a list of metrics as the Prometheus™ text exposition format |
| `prometheus.pushgatewayPath(job as string, grouping as map of string to string)` | `string` | Build the Pushgateway™ URL path for a job and a set of grouping labels |
| `prometheus.query(base as string, promql as string)` | `Result` | Run an instant query against `base` (a Prometheus server URL) via `/api/v1/query`, returning the parsed result set |
| `prometheus.queryRange(base as string, promql as string, start as string, end as string, step as string)` | `Result` | Run a range query against `base` via `/api/v1/query_range`, returning the parsed result matrix |

## Types

### `Sample`

One recorded series of a metric, keyed by its label set.

| Field | Meaning |
|---|---|
| `labels as map of string to string` | the label name/value pairs ({} for none) |
| `value as float` | the sample value (Counter / Gauge) |
| `count as float` | the observation count (Histogram / Summary) |
| `sum as float` | the observation sum (Histogram / Summary) |
| `buckets as list of float` | cumulative per-bucket counts (Histogram; parallel to Metric.buckets) |
| `observations as list of float` | the raw observed values (Summary; source for quantiles) |
| `timestamp as int` | an explicit millisecond timestamp, appended after the value when set |
| `hasTimestamp as bool` | whether `timestamp` is present and should be rendered |

### `Metric`

A metric family.

| Field | Meaning |
|---|---|
| `name as string` | the metric name (`[a-zA-Z_:][a-zA-Z0-9_:]*`) |
| `help as string` | the HELP text (empty to omit the HELP line) |
| `type as MetricType` | the metric type |
| `samples as list of Sample` | the recorded series |
| `buckets as list of float` | the histogram upper bounds (`le`), ascending; empty for other types |
| `quantiles as list of float` | the summary quantiles in `[0, 1]`, ascending; empty for other types |

### `Point`

One (timestamp, value) point of a query result series.

| Field | Meaning |
|---|---|
| `timestamp as float` | the sample time as a Unix timestamp (seconds) |
| `value as float` | the sample value |

### `Series`

One result series.

| Field | Meaning |
|---|---|
| `metric as map of string to string` | the series label set |
| `values as list of Point` | the series points |

### `Result`

A parsed query result set.

| Field | Meaning |
|---|---|
| `resultType as string` | "vector", "matrix", "scalar", or "string" (a string result whose text is not a number throws) |
| `series as list of Series` | the result series |

### `PrometheusError`

Every error of kind `"prometheus"`.

| Field | Meaning |
|---|---|
| `kind as string` | always `"prometheus"` |
| `message as string` | `prometheus: ` and the server's error, or what went wrong |
| `errorType as string` | a query error's `errorType` (`bad_data`, `execution`, `timeout`, ...); "" for another failure |
| `status as int` | the HTTP status of the query's answer; 0 when none |

### `MetricType`

A Prometheus metric type.

| Variant | Meaning |
|---|---|
| `Counter` | |
| `Gauge` | |
| `Histogram` | |
| `Summary` | |
