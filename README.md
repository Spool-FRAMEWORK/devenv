# spool-framework/devenv

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://www.apache.org/licenses/LICENSE-2.0)
[![Docker Compose](https://img.shields.io/badge/Docker%20Compose-2.x-2496ED.svg)](https://docs.docker.com/compose/)

**devenv** is the standalone observability stack for the [Spool](https://github.com/spool-framework) data engineering framework. It bundles **Grafana, Grafana Tempo, Grafana Loki, Prometheus and the OpenTelemetry Collector** with pre-built dashboards for every Spool module — ready to run with a single command.

Use this repository when you already have your own Kafka and PostgreSQL and only need to add observability to your Spool pipeline. If you want the full local development environment including infrastructure, use the `.docker/` directory inside each module instead.

---

## Table of Contents

- [What's included](#whats-included)
- [Quick Start](#quick-start)
- [Ports](#ports)
- [Connecting a Spool node](#connecting-a-spool-node)
- [Dashboards](#dashboards)
- [Architecture](#architecture)
- [License](#license)

---

## What's included

| Component | Image | Role |
|---|---|---|
| **Grafana** | `grafana/grafana:11.4.0` | Visualization — dashboards, traces explorer, log viewer |
| **Grafana Tempo** | `grafana/tempo:2.6.1` | Distributed trace storage and query backend |
| **Grafana Loki** | `grafana/loki:3.3.0` | Log aggregation backend |
| **Prometheus** | `prom/prometheus:v2.55.0` | Metrics storage and query backend |
| **OTel Collector** | `otel/opentelemetry-collector-contrib:0.123.0` | Metrics ingestion via OTLP, export to Prometheus |

All data sources are pre-configured in Grafana. All four module dashboards are pre-loaded in the **Spool** folder.

---

## Quick Start

```bash
git clone https://github.com/spool-framework/devenv.git
cd devenv
docker compose up
```

Grafana is available at **http://localhost:3000** (no login required).

---

## Ports

| Port | Service | Protocol | Description |
|---|---|---|---|
| `3000` | Grafana | HTTP | Dashboard UI |
| `4317` | Grafana Tempo | gRPC (OTLP) | Trace ingestion — point your Spool node here |
| `4318` | Grafana Tempo | HTTP (OTLP) | Trace ingestion (HTTP alternative) |
| `4319` | OTel Collector | gRPC (OTLP) | Metrics ingestion — point your Spool node here |
| `4320` | OTel Collector | HTTP (OTLP) | Metrics ingestion (HTTP alternative) |
| `3100` | Grafana Loki | HTTP | Log ingestion and query |
| `3200` | Grafana Tempo | HTTP | Tempo query API (used internally by Grafana) |
| `9090` | Prometheus | HTTP | Metrics query API (used internally by Grafana) |

---

## Connecting a Spool node

Configure your Spool node's OpenTelemetry exporter to point to this stack:

```yaml
# application.yml / SpoolNode configuration
otel:
  exporter:
    otlp:
      traces-endpoint: http://localhost:4317    # Tempo — gRPC
      metrics-endpoint: http://localhost:4319   # OTel Collector — gRPC
```

Or via JVM system properties:

```bash
java \
  -Dotel.exporter.otlp.traces.endpoint=http://localhost:4317 \
  -Dotel.exporter.otlp.metrics.endpoint=http://localhost:4319 \
  -jar your-spool-node.jar
```

Traces and metrics will appear in Grafana automatically once the node starts emitting telemetry.

---

## Dashboards

Four dashboards are pre-loaded in the **Spool** folder in Grafana. Each dashboard supports `$node`, `$source` and `$module` template variables for filtering across multiple active nodes or sources.

| Dashboard | Key metrics |
|---|---|
| **Spool — Crawler** | Capture rate, error breakdown by type, fetch latency (p50/p95/p99) |
| **Spool — Ingester** | Quarantine ratio, flush latency by policy, buffer size |
| **Spool — Janitor** | Inbox cleanup cycles, events republished, event age distribution |
| **Spool — Watchdog** | Module health state, mean downtime, aggregate node health percentage |

---

## Architecture

```
Spool node
    │
    ├─── traces (OTLP gRPC :4317) ──────────► Grafana Tempo ─────────► Grafana
    │
    ├─── metrics (OTLP gRPC :4319) ─────────► OTel Collector
    │                                               │
    │                                               └── Prometheus endpoint :8889
    │                                                         │
    │                                                    Prometheus ──────────► Grafana
    │
    └─── logs (OTLP HTTP :3100) ─────────────► Grafana Loki ──────────► Grafana
```

---

## License

Distributed under the [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0).
