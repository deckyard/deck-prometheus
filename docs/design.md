<!-- SPDX-License-Identifier: LGPL-3.0-only -->
<!-- SPDX-FileCopyrightText: Copyright (C) 2026 mplx <mplx@jennifer-lang.dev> -->

# Design

## Scope

The deck has two halves:

- **Exposition**: build counters, gauges, histograms and summaries
  (`counter`, `gauge`, `histogram`, `summary`), record observations
  (`observe`, `observeAt`), and `render` them in the Prometheus text format.
  `pushgatewayPath` builds the Pushgateway URL path for a job and grouping
  labels.
- **Retrieval**: `query` and `queryRange` read the Prometheus HTTP query API
  and return a parsed `Result`.

It deliberately leaves out:

- **Transport for exposition.** `render` returns a string; writing it to a
  textfile-collector file, POSTing it to a Pushgateway or serving it from a
  `/metrics` handler is a line of `fs`, `http` or `web` code, and the choice
  belongs to the program.
- **A global registry and automatic collectors.** The caller holds the metric
  set; there is no default registry and no process or runtime metrics.
- **OpenMetrics, protobuf and native histograms.** The classic text format is
  what every Prometheus version scrapes.
- **PromQL building or evaluation.** The server evaluates queries; results
  are read-only values.
- **The rest of the HTTP API** (series, labels, targets, rules, admin).

## Decisions

- **A deck, not a bundled module.** A client for one vendor's format and API
  belongs in the deck ecosystem. The deck is one of the bundled modules that
  `jennifer` moves out (M30.1 in its milestones, formerly M28.32) and keeps
  the module's surface, so migrating changes only the import.
- **Value semantics.** A `Metric` is a plain struct; `observe` returns the
  updated metric. To keep this cheap, the observation core changes the
  caller's copy instead of copying every series, which is why a metric kept
  in a function's local variable is fast and a top-level one is copied on
  each call.
- **Counters and gauges are set, not incremented.** `observe` on a counter
  or gauge replaces the sample for that label set (last write wins); the
  caller keeps the count. Histograms and summaries accumulate.
- **Histogram buckets** are sorted ascending, a repeated bound dropped, and
  an `+Inf` bucket always rendered; counts are cumulative.
- **Exact summary quantiles.** A summary keeps every observation and computes
  nearest-rank quantiles at render time, rather than the streaming,
  bounded-error estimators of the reference clients: simpler and exact, at
  the cost of memory and a sort per render. Long-running processes should
  prefer a histogram.
- **Series are found by a scan** of the metric's label sets, the last series
  touched first. Fine for hundreds of series; thousands are slow.
- **Strict exposition.** Metric names must match
  `[a-zA-Z_:][a-zA-Z0-9_:]*`, label names `[a-zA-Z_][a-zA-Z0-9_]*` without the
  reserved `__` prefix; a histogram cannot take `le` nor a summary
  `quantile`. `render` checks names again, because `Metric` and `Sample`
  fields can be set directly and a name with a newline would inject lines,
  and refuses a metric name given twice. Label values escape `\`, `"` and
  newline; HELP text escapes `\` and newline.
- **Pushgateway paths that survive the gateway.** Grouping keys are sorted
  for a deterministic path. A value with a `/` (which the Pushgateway splits
  even as `%2F`) or an empty one uses the gateway's base64 form.
- **Query results are floats.** `Point` holds a float timestamp and value.
  A Jennifer float holds no NaN or infinity, so such a value (a division by
  zero, `histogram_quantile` over an empty bucket) is an error naming it, as
  is a native-histogram point; a `"string"` result must be a number.
- **Same result shape as the InfluxDB deck.** `Result` / `Series` mirror the
  read side of `@jennifer/influxdb`.

## Errors

`PrometheusError` (kind `"prometheus"`) is the one struct for every error of
that kind, with `errorType` (the server's `errorType`, such as `bad_data`,
`""` otherwise) and `status` (the HTTP status of a query reply, 0 for a local
check). It is thrown for:

- every exposition rule above, and an empty Pushgateway job or a grouping
  label that is invalid, reserved or `job`;
- a query reply whose `status` is not `success`, with the server's `error`;
- a reply that is not JSON, or whose shape the parser does not expect, and
  values it cannot hold;
- a `base` URL with a query or fragment.

Transport failures are not caught: they come from `http` as it throws them
(`"timeout"`, `"closed"`, `"limit"`, and on interpreters with transport error
kinds, M28.65, a `net.NetError` of kind `"net"`). A reply past `json`'s
decoding caps is passed on as kind `"limit"` (M28.68), as in the bundled
copy.

## Security and limits

- Server text in an error message has control characters (except tab)
  replaced by spaces, so it cannot forge log lines, and is cut at 512
  characters.
- The exposition checks make `render` output safe to serve even when names
  or labels come from elsewhere: no value can add a line or a label.
- The query client sends no credentials; a Prometheus behind authentication
  needs a proxy or a caller-built request. The PromQL expression is
  percent-encoded into the URL.
- Response size and deadlines are `http`'s defaults; JSON decoding is under
  `json`'s caps.

## Testing and validation

`src/prometheus_test.j` pins the rendered text of every metric type (labels,
sorting, upserts, histogram buckets, summary quantiles, timestamps), the name
and label checks, the Pushgateway path including its base64 form, and the
result parser over canned vector, matrix, scalar and error replies, as well
as every parse failure and the cleaned server error. `query` and
`queryRange` run end to end against an `httpd` server on a loopback port,
which also shows that the PromQL encoding round-trips.

The module's histogram and summary work (M23.7) was validated against an
independent reference parser of the text format when it was added. The
tests never reach a real Prometheus; `examples/exposition.j` prints a
rendered metric set and needs no server.

## Known limitations and open questions

- No authentication on queries.
- A `"string"` result that is not a number cannot be returned.
- Summaries keep every observation; there is no age window.

## Dependencies on the interpreter

- The floor is `>=0.27.0`, for the UTF-8 codec of `encoding.encode` and
  `strings.replicate`; the bundled module is deprecated from 0.26.0 (M30.1).
- Thrown error structs are classified by kind in `testing.assertThrows` and
  accepted by lint's L104 (M28.50, M28.51).
- The deck imports `http`, which needs `net`, so it loads only on the default
  `jennifer` binary, even for exposition alone.
