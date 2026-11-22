# spool-framework/devenv

[![License](https://img.shields.io/badge/License-AGPL%203.0-blue.svg)](LICENSE)
[![Docker Compose](https://img.shields.io/badge/Docker%20Compose-2.x-2496ED.svg)](https://docs.docker.com/compose/)

**devenv** is the standalone observability stack for the [Spool](https://github.com/spool-framework) data engineering framework. It bundles **Grafana, Grafana Tempo, Grafana Loki, Prometheus and the OpenTelemetry Collector** with pre-built dashboards for every Spool module — ready to run with a single command.

Use this repository when you already have your own Kafka and PostgreSQL and only need to add observability to your Spool pipeline. If you want the full local development environment including infrastructure, use the `.docker/` directory inside each module instead.

---

## Table of Contents

- [What's included](#whats-included)
- [Quick Start](#quick-start)
- [Ports](#ports)
- [Connecting a Spool node](#connecting-a-spool-node)
- [Authenticated backends](#authenticated-backends)
- [Identity provider](#identity-provider)
- [Dashboards](#dashboards)
- [Quarantine](#quarantine)
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
| **Keycloak** | `quay.io/keycloak/keycloak:26.0.7` | OpenID Connect provider for the HTTP entries of a node, with one user per role |

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
| `4317` | Grafana Tempo | gRPC (OTLP) | Trace ingestion (gRPC alternative, not used by Spool nodes) |
| `4318` | Grafana Tempo | HTTP (OTLP) | Trace ingestion — used by Spool nodes, path `/v1/traces` |
| `4319` | OTel Collector | gRPC (OTLP) | Metrics ingestion (gRPC alternative, not used by Spool nodes) |
| `4320` | OTel Collector | HTTP (OTLP) | Metrics ingestion — used by Spool nodes, path `/v1/metrics` |
| `3100` | Grafana Loki | HTTP (OTLP) | Log ingestion — used by Spool nodes, path `/otlp/v1/logs` — and log query |
| `3200` | Grafana Tempo | HTTP | Tempo query API (used internally by Grafana) |
| `9090` | Prometheus | HTTP | Metrics query API (used internally by Grafana) |
| `8180` | Keycloak | HTTP | OpenID Connect issuer `http://localhost:8180/realms/spool`, and its admin console |

---

## Connecting a Spool node

A Spool node exports traces, logs and metrics over **OTLP/HTTP**, so it connects to the HTTP ports of this stack and not to the gRPC ones. Each signal has its own endpoint, with its full path:

| Signal | Environment variable | Default |
|---|---|---|
| Traces | `OTEL_TRACES_ENDPOINT` | `http://localhost:4318/v1/traces` |
| Logs | `OTEL_LOGS_ENDPOINT` | `http://localhost:3100/otlp/v1/logs` |
| Metrics | `OTEL_METRICS_ENDPOINT` | `http://localhost:4320/v1/metrics` |
| Service name | `SERVICE_NAME` | the name given in code |

The defaults already match this stack, so a node running on the same machine needs no configuration. To point it somewhere else, set the variables or use the runtime builder:

```java
OpenTelemetryConfiguration configuration = OpenTelemetryConfiguration.builder()
        .serviceName("spool")
        .tracesEndpoint("http://localhost:4318/v1/traces")
        .logsEndpoint("http://localhost:3100/otlp/v1/logs")
        .metricsEndpoint("http://localhost:4320/v1/metrics")
        .build();

SpoolRuntime.builder()
        .OpenTelemetryConfiguration(configuration)
        .withNodeFromDSL("/descriptors/node.yaml")
        .build();
```

An endpoint set in code wins over its environment variable, which wins over the default.

Traces, logs and metrics will appear in Grafana automatically once the node starts emitting telemetry.

---

## Authenticated backends

A backend that requires credentials, such as Grafana Cloud or a collector behind an authenticating proxy, needs headers on every export request. Headers are set per exporter, with the same options for traces, logs and metrics.

### Without code

Set the standard OpenTelemetry variable. Values are percent-encoded, so a space is written `%20`:

```bash
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Basic%20<base64 of instance-id:token>"
```

Use `OTEL_EXPORTER_OTLP_TRACES_HEADERS`, `OTEL_EXPORTER_OTLP_LOGS_HEADERS` or `OTEL_EXPORTER_OTLP_METRICS_HEADERS` to set headers for one signal only. Several headers are separated by commas, as in `Name=value,Other=value`.

### In code

```java
OpenTelemetryConfiguration configuration = OpenTelemetryConfiguration.builder()
        .serviceName("spool")
        .tracesEndpoint("https://<gateway>/v1/traces")
        .logsEndpoint("https://<gateway>/v1/logs")
        .metricsEndpoint("https://<gateway>/v1/metrics")
        .headerFromEnvironment("Authorization", "GRAFANA_OTLP_AUTHORIZATION")
        .compression("gzip")
        .build();
```

`headerFromEnvironment` names the environment variable that holds the value, so the secret never appears in code or in a descriptor. Here `GRAFANA_OTLP_AUTHORIZATION` holds the whole header value, for example `Basic <base64 of instance-id:token>`. If the variable is not set the node fails to start, and the message names the variable and the header, never the value.

### Precedence

When the same header is given more than once, the last one in this list wins. Header names are case-insensitive.

1. `OTEL_EXPORTER_OTLP_HEADERS`
2. `OTEL_EXPORTER_OTLP_<SIGNAL>_HEADERS`
3. `header(name, value)` on the builder
4. `headerFromEnvironment(name, variable)` on the builder

Use `header(name, value)` only for values that are not secret, such as a tenant id in `X-Scope-OrgID`.

### Other options

`compression("gzip")` compresses the requests, and `timeout(Duration)` sets how long an export may take. Both apply to the three signals. To give one signal its own settings, pass an `OTELExporterSettings` to `traces(...)`, `logs(...)` or `metrics(...)`: what it sets wins over the values shared by the three, and its headers are added to the shared ones.

---

## Identity provider

Every HTTP entry of a node, except its health, is authenticated by a provider declared in the descriptor (ADR-013 in the Spool Linear project). Keycloak plays that provider locally. It imports the `spool` realm from `keycloak/spool-realm.json`:

| User | Password | Roles |
|---|---|---|
| `reader` | `reader` | `quarantine-reader` |
| `resolver` | `resolver` | `quarantine-reader`, `quarantine-resolver` |
| `bypasser` | `bypasser` | `quarantine-reader`, `quarantine-resolver`, `quarantine-bypasser` |

These passwords, and `admin` / `admin` for the admin console, only exist in this local stack. Never reuse the realm file anywhere else.

Point a node at it with an `OIDC` provider. The keys are found through the issuer's discovery document:

```yaml
authentication:
  people:
    type: OIDC
    configuration:
      issuer: http://localhost:8180/realms/spool
      audience: spool-janitor
      rolesClaim: realm_access.roles
```

Ask for a token as one of the users and send it as `Authorization: Bearer <token>`:

```bash
curl -s -d grant_type=password -d client_id=spool-janitor -d username=reader -d password=reader \
  http://localhost:8180/realms/spool/protocol/openid-connect/token
```

The `spool-janitor` client adds itself to the audience of every token, so the `audience` above matches.

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

## Quarantine

An envelope in quarantine is one that will not be processed as it is. It stays in the inbox in the `QUARANTINED` state, with its payload intact, and its key stays taken, so the same payload is rejected as a duplicate if it is captured again.

### How an envelope gets there

- **Its retries run out.** The janitor republishes an envelope that has been captured for longer than its threshold. After `maxRetries` attempts, 3 by default and not settable from a descriptor, it moves it to quarantine.
- **The ingester validation flags it.** A critical violation of the validator sends the envelope to the quarantine store and publishes an `EnvelopeQuarantined` event with the violations, and the janitor moves the envelope. An ingester built from a descriptor has no quarantine store today, so it cannot store the record and the event is not published: there, an envelope reaches quarantine by the first way.

### Who is told

| Way in | Log of the janitor | Metric `reason` | Event |
|---|---|---|---|
| Validation | `INFO` "Quarantined N envelopes" | `quarantine` | `EnvelopeQuarantined`, with the violations |
| Retries run out | `ERROR` "Quarantined Envelope … after N attempts" | `retries_exhausted` | none |

Both add to `spool_janitor_records_cleaned_total`, which the **Records cleaned / s** and **Records cleaned (total)** panels of the **Spool — Janitor** dashboard show by `reason`. The metric holds four reasons:

| `reason` | What it counts |
|---|---|
| `expired` | Persisted envelopes deleted after `millisecondsTTL` |
| `quarantine` | Envelopes moved to quarantine after a validation failure |
| `retries_exhausted` | Envelopes moved to quarantine after their retries ran out |
| `quarantine_expired` | Quarantined envelopes deleted after `millisecondsQuarantineTTL` |

The `SpoolEnvelopesQuarantined` alert, in `alerts.yml`, fires when either of the two quarantine reasons grows. This stack has no Alertmanager, so the alert shows at http://localhost:9090/alerts and in Grafana but is not sent anywhere. Connect Prometheus to an Alertmanager, or create the alert in Grafana with a contact point, to be notified.

`alerts.test.yml` checks when the alert fires and when it does not, with made-up series. Run it after changing the rule:

```bash
docker run --rm -v "$PWD:/work" -w /work --entrypoint promtool prom/prometheus:v2.55.0 test rules alerts.test.yml
```

### How long it stays

Nothing removes it by default. To delete what has been in quarantine for longer than a time, set `millisecondsQuarantineTTL` in the janitor:

```yaml
- janitor:
    id: synthea-janitor
    configuration:
      millisecondsTTL: 86400000
      millisecondsQuarantineTTL: 604800000
```

Deleting is final: the payload is lost. Leave it unset until someone has decided what should happen to those envelopes.

### Requeue or discard

`QuarantineReleaser` lists what is in quarantine and deals with it:

```java
QuarantineReleaser releaser = QuarantineReleaser.using(inboxReader, inboxUpdater, inboxEnvelopeRemover, eventPublisher);

Collection<Envelope> quarantined = releaser.quarantined();
ReleaseResult requeued = releaser.requeue(keys);
ReleaseResult discarded = releaser.discard(keys);
```

- `requeue` returns the envelopes to the inbox with their retries at zero and announces them, so the ingester processes them again.
- `discard` deletes them, which frees their keys so the same payloads can be captured again.
- Both only act on envelopes that are in quarantine. A key that is not, or that does not exist, is skipped and appears in `ReleaseResult.skipped()`.

There is no command for it yet: it is an API.

---

## Architecture

```
Spool node
    │
    ├─── traces (OTLP HTTP :4318) ──────────► Grafana Tempo ─────────► Grafana
    │
    ├─── metrics (OTLP HTTP :4320) ─────────► OTel Collector
    │                                               │
    │                                               └── Prometheus endpoint :8889
    │                                                         │
    │                                                    Prometheus ──────────► Grafana
    │
    └─── logs (OTLP HTTP :3100) ─────────────► Grafana Loki ──────────► Grafana
```

---

## License

Distributed under the [GNU Affero General Public License v3.0](LICENSE).
