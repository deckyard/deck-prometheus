# @jennifer/prometheus

[![test](https://github.com/jennifer-language/deck-prometheus/actions/workflows/test.yml/badge.svg)](https://github.com/jennifer-language/deck-prometheus/actions/workflows/test.yml)
[![Documentation](https://img.shields.io/badge/docs-github%20pages-blue)](https://jennifer-language.github.io/deck-prometheus/)
[![release](https://img.shields.io/github/v/release/jennifer-language/deck-prometheus)](https://github.com/jennifer-language/deck-prometheus/releases/latest)

A [Jennifer](https://jennifer-lang.dev/) deck for
[Prometheus™](https://prometheus.io/) metrics: build counters, gauges,
histograms and summaries, render them in the text exposition format (for a
`/metrics` handler, the node_exporter textfile collector or a Pushgateway™),
and query a Prometheus server's HTTP API.

It was Jennifer's bundled `prometheus.j` module until 0.27; the surface is the same,
so moving over changes only the import:

```jennifer
import "@jennifer/prometheus/" as prometheus;   # was: import "prometheus.j" as prometheus;
```

**[Documentation](https://jennifer-language.github.io/deck-prometheus/)**

## Install

```sh
jvc add @jennifer/prometheus
```

This vendors the deck to `vendor/<scope>/prometheus/`. Without `jvc`, unpack the
`src/` subtree of a release archive (`<scope>-prometheus-<version>.tar.gz`) there.

## Usage

```jennifer
import "@jennifer/prometheus/" as prometheus;

def m as prometheus.Metric init prometheus.counter("http_requests_total", "Total HTTP requests");
$m = prometheus.observe($m, {"method": "get", "code": "200"}, 1.0);
io.printf("%s", prometheus.render([$m]));
```

## Requirements

Jennifer 0.27.0 or newer, the default `jennifer` binary: the module imports
`http` (`net`), so `jennifer-tiny` cannot load it.

## Development

```sh
tools/check-all.sh                   # the full gate, as CI runs it
jennifer test src/prometheus_test.j      # the test overlay
tools/check-docs.sh                  # docs against the code
tools/pack.sh dist                   # build the release archive
```

## Licence

LGPL-3.0-only, see [LICENSE](LICENSE). Copyright (C) 2026 mplx &lt;mplx@jennifer-lang.dev&gt;.

`tools/check-headers.sh` checks every source file's SPDX header against
`deck.toml`.

## Trademarks

Prometheus™ is a trademark of The Linux Foundation. Other product and company
names may be trademarks of their respective owners. They are used only to
identify the products this deck works with. This deck is an independent
project, not affiliated with, sponsored or endorsed by The Linux Foundation.
