# LGTM

A compose based LGTM observability stack.
LGTM stands for

* L — Loki (for logs)
* G — Grafana (for visualization and correlation)
* T — Tempo (for distributed traces)
* M — Prometheus / Mimir (for metrics — with Prometheus handling collection/storage and Mimir often used for long-term scalable metrics)

## Quick start

Clone the repository.

```bash
docker compose up -d
```

Open Grafana http://localhost:4000 with
* Username: `admin`
* Password: `admin`

Then configure your application(s) to report to the respective endpoiints.

## Services

| Service | Image | Host port | Purpose |
|---|---|---|---|
| **collector** | [`otel/opentelemetry-collector-contrib`](https://github.com/open-telemetry/opentelemetry-collector-contrib) | 4317 (gRPC), 4318 (HTTP) | Receives OTLP; fans out to all backends |
| **prometheus** | [`prom/prometheus`](https://prometheus.io/docs/introduction/overview/) | | Scrapes collector metrics endpoint; time-series storage |
| **loki** | [`grafana/loki`](https://grafana.com/docs/loki/latest/) | | Stores structured log events via OTLP HTTP |
| **tempo** | [`grafana/tempo`](https://grafana.com/docs/tempo/latest/) | | Stores distributed traces via OTLP gRPC |
| **grafana** | [`grafana/grafana`](https://grafana.com/docs/grafana/latest/) | 4000 | Visualization — Explore and dashboards across all three signals |

### OpenTelemetry Collector

The collector (`otel/collector.yaml`) is the single ingestion point.
It runs three independent pipelines:

| Pipeline | Receiver | Exporters |
|---|---|---|
| metrics | `otlp` | `prometheus` (scrape endpoint), `debug` |
| logs | `otlp` | `otlphttp/loki`, `debug` |
| traces | `otlp` | `otlp_grpc/tempo`, `debug` |

The `debug` exporter prints all received data to the collector's stdout — useful for verifying the pipeline without opening Grafana.

**Docs:** [OTel Collector](https://opentelemetry.io/docs/collector/) · [Contrib receivers/exporters](https://github.com/open-telemetry/opentelemetry-collector-contrib)

### Prometheus

Prometheus scrapes the collector's `/metrics` endpoint at `collector:8889` every 15 seconds.

**Docs:** [Prometheus](https://prometheus.io/docs/prometheus/latest/getting_started/)

### Loki

Loki receives log events from the collector over OTLP HTTP (`http://loki:3100/otlp`).
No log-agent sidecar is needed — the OTel Collector pushes directly to Loki's native OTLP endpoint (available since Loki 3.x).

**Docs:** [Loki](https://grafana.com/docs/loki/latest/) · [Loki OTLP ingest](https://grafana.com/docs/loki/latest/send-data/otel/)

### Tempo

Tempo receives traces from the collector over OTLP gRPC `tempo:4317`.

**Docs:** [Tempo](https://grafana.com/docs/tempo/latest/) · [Tempo OTLP ingest](https://grafana.com/docs/tempo/latest/configuration/network/receiver/)

### Grafana

Grafana starts with three datasources pre-provisioned via `grafana/provisioning/datasources/`:

| Datasource | UID | URL |
|---|---|---|
| Prometheus | `prometheus` | `http://prometheus:9090` |
| Loki | `loki` | `http://loki:3100` |
| Tempo | `tempo` | `http://tempo:3200` |

Tempo is configured with `tracesToLogsV2` linked to Loki, so you can jump from a trace span directly to the correlated log lines.

**Docs:** [Grafana](https://grafana.com/docs/grafana/latest/) · [Explore](https://grafana.com/docs/grafana/latest/explore/)
