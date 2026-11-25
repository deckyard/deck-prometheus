<!-- SPDX-License-Identifier: LGPL-3.0-only -->
<!-- SPDX-FileCopyrightText: Copyright (C) 2026 mplx <mplx@jennifer-lang.dev> -->

# @deckyard/prometheus

Metrics exposition and query.

Add it with `jvc add @deckyard/prometheus` and import it with `import "@deckyard/prometheus/" as prometheus;`. A **Prometheus™** module in
two halves. **Exposition** builds a metric set and renders the Prometheus text
format - pure text over `strings` / `maps` / `lists` / `convert`, and
transport-agnostic: write the string to a `*.prom` file for the node_exporter
textfile collector, POST it to a Pushgateway™, or serve it from a `/metrics`
handler. **Retrieval** is a read client for Prometheus's HTTP query API, built on
the [`http`](https://jennifer-lang.dev/modules/http.html) module + [`json`](https://jennifer-lang.dev/libraries/json.html).

The module imports `http`, which needs `net`, so it loads only on the default
**`jennifer`** binary, even for a program that only renders: `jennifer-tiny`
refuses it at load time.

```jennifer
import "@deckyard/prometheus/" as prometheus;

def m as prometheus.Metric init prometheus.counter("http_requests_total",
    "Total HTTP requests");
$m = prometheus.observe($m, {"method": "get", "code": "200"}, 42.0);
io.printf("%s", prometheus.render([$m]));
# # HELP http_requests_total Total HTTP requests
# # TYPE http_requests_total counter
# http_requests_total{code="200",method="get"} 42.0
```

Runnable: [`examples/exposition.j`](https://github.com/deckyard/deck-prometheus/blob/main/examples/exposition.j).

## Exposition

Build metrics, record samples, render the text format. Every builder is
value-semantic (returns a new `Metric`), so a metric set is assembled by
reassignment.

| Call / type                          | Notes                                                                 |
| ------------------------------------ | --------------------------------------------------------------------- |
| `prometheus.Metric`                  | `name`, `help`, `type` (`Counter` / `Gauge` / `Histogram` / `Summary`), `samples`, `buckets`, `quantiles`. |
| `prometheus.Sample`                  | `labels` (map), `value` (float), plus `count` / `sum` / `buckets` / `observations` / `timestamp` / `hasTimestamp` for the aggregate types. |
| `prometheus.counter(name, help)`     | A new counter metric; throws on an invalid name.                      |
| `prometheus.gauge(name, help)`       | A new gauge metric; throws on an invalid name.                        |
| `prometheus.histogram(name, help, buckets)` | A new histogram; `buckets` is a `list of float` of upper bounds (`le`); sorted, repeats dropped. |
| `prometheus.summary(name, help, quantiles)` | A new summary; `quantiles` is a `list of float` in `[0, 1]`; sorted for you. |
| `prometheus.observe(metric, labels, value)` | Record an observation; throws on an invalid or reserved label name. |
| `prometheus.observeAt(metric, labels, value, timestampMs)` | As `observe`, plus an explicit millisecond timestamp. |
| `prometheus.render(metrics)`         | Render a `list of Metric` as the text exposition format.              |
| `prometheus.pushgatewayPath(job, grouping)` | Build the `/metrics/job/<job>/<k>/<v>/...` Pushgateway path (pure string). |

For a `Counter` / `Gauge`, `observe` **upserts**: a sample with an equal label
set is replaced (last write wins), so re-observing the same series updates its
value rather than duplicating the line. For a `Histogram` / `Summary`, `observe`
**accumulates** (see below). `render` sorts label keys, so output is
deterministic regardless of the order labels were inserted.

### Histograms and summaries

A **histogram** buckets observations by an upper bound (`le`). Each `observe`
increments the count, adds to the sum, and increments every cumulative bucket
whose bound is `>=` the value; `render` emits, for a metric named `x`:
`x_bucket{le="..."}` (cumulative counts, ascending, `le="+Inf"` last), `x_sum`,
and `x_count`, under a `# TYPE x histogram` header.

```jennifer
def h as prometheus.Metric init prometheus.histogram(
    "http_request_duration_seconds", "Request latency", [0.1, 0.5, 1.0]);
$h = prometheus.observe($h, {"method": "get"}, 0.3);
$h = prometheus.observe($h, {"method": "get"}, 0.05);
$h = prometheus.observe($h, {"method": "get"}, 2.0);
io.printf("%s", prometheus.render([$h]));
# # TYPE http_request_duration_seconds histogram
# http_request_duration_seconds_bucket{le="0.1",method="get"} 1.0
# http_request_duration_seconds_bucket{le="0.5",method="get"} 2.0
# http_request_duration_seconds_bucket{le="1.0",method="get"} 2.0
# http_request_duration_seconds_bucket{le="+Inf",method="get"} 3.0
# http_request_duration_seconds_sum{method="get"} 2.35
# http_request_duration_seconds_count{method="get"} 3.0
```

A **summary** reports quantiles. Each `observe` increments the count, adds to
the sum, and retains the observed value; `render` emits `x{quantile="..."}`
(computed from the retained observations by the nearest-rank method,
`rank = ceil(q * n)`), `x_sum`, and `x_count`, under `# TYPE x summary`. Each
label set accumulates its own count / sum / buckets / observations
independently.

### Timestamps

`observeAt(metric, labels, value, timestampMs)` records a sample carrying an
explicit millisecond Unix timestamp; `render` appends it after the value
(`metric value timestamp`), the optional third field the exposition format
allows. Plain `observe` records no timestamp.

### Pushing to a Pushgateway

`pushgatewayPath(job, grouping)` is a pure string helper that builds the
Pushgateway URL path from a job name and a `map of string to string` of grouping
labels: `/metrics/job/<job>/<k1>/<v1>/...`, grouping keys sorted for a
deterministic path. Values are percent-encoded; one with a `/` (which the
Pushgateway splits even as `%2F`) or an empty one uses its base64 form,
`/<k>@base64/<url-safe base64>` (`=` for empty). POST the `render` output to
`base + pushgatewayPath(job, grouping)` via the
[`http`](https://jennifer-lang.dev/modules/http.html) module. An empty job, or
a grouping label name that is invalid, reserved or `job`, throws.

### Strictness

These rules are checked, and a violation throws a `PrometheusError` (kind
`"prometheus"`, see the cheatsheet). A parameter typed `Error` does not accept
the struct: catch it untyped, or type a helper's parameter as
`prometheus.PrometheusError`.

- A metric name must match `[a-zA-Z_:][a-zA-Z0-9_:]*`; a label name must match
  `[a-zA-Z_][a-zA-Z0-9_]*` and not start with `__` (reserved). A histogram
  cannot take an `le` label nor a summary a `quantile` one; they add those.
- `render` checks names again (the struct fields can be set directly, and a
  name with a newline would inject lines) and refuses a metric name given
  twice.
- Label values escape `\`, `"`, and newline; `# HELP` text escapes `\` and
  newline. An empty `help` omits the `# HELP` line.
- Values cannot be NaN or infinite: Jennifer floats do not hold them.

### Getting the text to Prometheus

`render` returns a plain string; delivery is your choice:

- **Textfile collector** - write it to a `*.prom` file (via `fs`) in
  node_exporter's textfile directory.
- **Pushgateway** - POST it with the [`http`](https://jennifer-lang.dev/modules/http.html) module.
- **Scrape endpoint** - serve it from a `/metrics` handler (e.g. the
  [`web`](https://jennifer-lang.dev/modules/web.html) framework over `httpd`).

## Retrieval

A read client for the HTTP query API. Both return a `Result`.

| Call / type                                       | Notes                                             |
| ------------------------------------------------- | ------------------------------------------------- |
| `prometheus.Result`                               | `resultType` + `series` (a `list of Series`).     |
| `prometheus.Series`                               | `metric` (label map) + `values` (a `list of Point`). |
| `prometheus.Point`                                | `timestamp` (float, Unix seconds) + `value` (float). |
| `prometheus.query(base, promql)`                  | Instant query (`/api/v1/query`) -> `Result`.      |
| `prometheus.queryRange(base, promql, start, end, step)` | Range query (`/api/v1/query_range`) -> `Result`. |

`base` is the server URL (e.g. `"http://localhost:9090"`). `start` / `end` are
RFC 3339 or Unix-timestamp strings; `step` is a duration (`"15s"`) or a seconds
string. An instant query returns a `"vector"` (one `Point` per series); a range
query returns a `"matrix"` (many `Point`s per series). A server-reported query
error throws a `PrometheusError` (kind `"prometheus"`, with the server's
`errorType` such as `bad_data` and the HTTP `status`), its text cut to 512 characters
with control characters replaced, as does a response the parser cannot read:
a `NaN` or `±Inf` value (which Prometheus returns for a division by zero, and
a Jennifer float cannot hold), a native-histogram point, a `"string"` result
that is not a number. A reply nested past `json`'s decoding caps is kind
`"limit"`, and transport failures come from `http` as it throws them (a
socket failure is a `net.NetError`, kind `"net"`). `base` may end in slashes
but cannot have a query or fragment.

```jennifer
def r as prometheus.Result init prometheus.query("http://localhost:9090", "up");
for (def s in $r.series) {
    io.printf("%s = %f\n", $s.metric["instance"], $s.values[0].value);
}
```

## Testing

The pure exposition logic - name / label validation, value and HELP escaping,
label-key sorting, the counter/gauge upsert, histogram bucket accumulation
(with pinned `_bucket` / `_sum` / `_count` output), summary quantiles,
timestamps, and the Pushgateway path - is unit-tested in the overlay
(`src/prometheus_test.j`), alongside the result parser against canned
vector / matrix / scalar / error responses. The networked `query` /
`queryRange` path is covered end to end against an `httpd` server on a
loopback port standing in for Prometheus, which also proves the PromQL™ URL
encoding round-trips.

## Out of scope

- **No registry / auto-collection.** The caller holds and assembles the metric
  set; there is no global default registry or process/Go collectors.
- **Summary quantiles are exact over the retained observations**, not the
  streaming, bounded-error estimators the reference clients use - simpler and
  more accurate, at the cost of keeping every observed value per series, and
  sorting them on every `render`. For a long-running process, prefer a
  histogram.
- **Series are found by a scan.** `observe` compares the label set with each
  series of the metric, so adding the Nth series costs N comparisons: fine for
  hundreds of series, slow (seconds) for thousands.
- **Keep a metric in a function's local variable.** `observe` updates the
  metric it is given without copying when the variable is local; a top-level
  variable passed to a module function is copied, every series with it.
- **Query results are read-only values.** No PromQL building or evaluation - the
  server does that.

## Moving from `@jennifer/prometheus`

Up to 0.4.0 this deck was `@jennifer/prometheus`. Change the `[decks]` entry to
`"@deckyard/prometheus" = "^0.5.0"` and the imports:

```jennifer
import "@deckyard/prometheus/" as prometheus;
```

## See also

- [http.md](https://jennifer-lang.dev/modules/http.html) - the client transport the retrieval half builds on.
- [json.md](https://jennifer-lang.dev/libraries/json.html) - the query-response decoder.
- [modules/index.md](https://jennifer-lang.dev/modules/index.html) - the module catalog and import rules.

## Trademarks

Prometheus™ is a trademark of The Linux Foundation. Other product and company
names may be trademarks of their respective owners. They are used only to
identify the products this deck works with. This deck is an independent
project, not affiliated with, sponsored or endorsed by The Linux Foundation.
