# Telemetry

`demo/midgard-node` exposes scrapeable Prometheus metrics from the Midgard node
process when it is started with monitoring enabled.

This document focuses on application metrics, tracing, and telemetry semantics
for the demo node. The demo also ships a local/container observability stack
through Docker Compose: Prometheus, Grafana, Loki, Promtail, Tempo, and
cAdvisor. The stack configuration lives under `demo/midgard-node/*`.

## Endpoints

- Node API: `http://localhost:3000` by default, controlled by `PORT`.
- Node metrics: `GET /metrics` on the OpenTelemetry Prometheus exporter port.
- Prometheus UI: `http://localhost:9090`
- Grafana UI: `http://localhost:3001`
- Loki: `http://localhost:3100`
- Tempo: `http://localhost:3200`
- cAdvisor: `http://localhost:8080`

Example local scrape target:

- Midgard node: `http://localhost:9464/metrics` when
  `PROM_METRICS_PORT=9464`

Prometheus scrapes the node through the `midgard_nodes` job at
`midgard-node:9464` inside the Docker network.

## Implementation Locations

Node telemetry setup lives in:

- `demo/midgard-node/src/commands/listen.ts`
- `demo/midgard-node/src/services/config.ts`

Application metrics live in:

- `demo/midgard-node/src/commands/listen.ts`
- `demo/midgard-node/src/fibers/block-commitment.ts`
- `demo/midgard-node/src/fibers/monitor-mempool.ts`
- `demo/midgard-node/src/fibers/tx-queue-processor.ts`
- `demo/midgard-node/src/transactions/state-queue/merge-to-confirmed-state.ts`

Observability stack configuration lives in:

- `demo/midgard-node/docker-compose.yaml`
- `demo/midgard-node/prometheus.yml`
- `demo/midgard-node/grafana/*`
- `demo/midgard-node/loki-config.yaml`
- `demo/midgard-node/promtail-config.yaml`
- `demo/midgard-node/tempo.yaml`

## Metrics Endpoint Env

- `PROM_METRICS_PORT`: Prometheus exporter port; defaults to `9464`.
- `OLTP_EXPORTER_URL`: OTLP HTTP trace endpoint; defaults to
  `http://0.0.0.0:4318/v1/traces`.

The metrics endpoint is separate from the node API port.

## Application Metrics

Counters are exported by the OpenTelemetry Prometheus exporter with a `_total`
suffix.

| Metric                         | Type    | Labels | Meaning                                                                     | Default usage                         |
| ------------------------------ | ------- | ------ | --------------------------------------------------------------------------- | ------------------------------------- |
| `tx_count_total`               | Counter | none   | Submitted transactions accepted by `POST /submit`.                          | Dashboard throughput signal.          |
| `tx_queue_size`                | Gauge   | none   | Size of the in-memory transaction queue before processing.                   | Dashboard queue/backlog signal.       |
| `mempool_tx_count`             | Gauge   | none   | Current number of transactions in `MempoolDB`.                              | Dashboard mempool signal.             |
| `commit_block_count_total`     | Counter | none   | Number of committed Midgard blocks.                                         | Dashboard block production signal.    |
| `commit_block_tx_count_total`  | Counter | none   | Number of transaction requests included in committed blocks.                 | Dashboard inclusion throughput signal. |
| `commit_block_num_tx_count`    | Gauge   | none   | Transaction request count in the latest committed block.                     | Dashboard latest-block context.       |
| `block_total_user_events_count` | Gauge  | none   | Deposit, withdrawal, and transaction-order event count in the latest block.  | Dashboard latest-block context.       |
| `total_tx_size`                | Gauge   | none   | Total event size for the latest committed block.                             | Dashboard latest-block context.       |
| `merge_block_count_total`      | Counter | none   | Number of blocks merged into confirmed state.                                | Dashboard merge throughput signal.    |

## Infrastructure Metrics

The local Docker Compose stack also scrapes metrics that do not originate inside
the Midgard node code:

| Metric                                      | Source                   | Meaning                                      | Default usage                    |
| ------------------------------------------- | ------------------------ | -------------------------------------------- | -------------------------------- |
| `up{job="midgard_nodes"}`                   | Prometheus target health | Scrape health for the Midgard node exporter. | Availability and scrape context. |
| `container_memory_usage_bytes`              | cAdvisor                 | Container memory usage.                      | Grafana container dashboard.     |
| `container_cpu_user_seconds_total`          | cAdvisor                 | Container CPU usage counter.                 | Grafana container dashboard.     |
| `container_network_receive_bytes_total`     | cAdvisor                 | Container receive traffic counter.           | Grafana container dashboard.     |
| `container_network_transmit_bytes_total`    | cAdvisor                 | Container transmit traffic counter.          | Grafana container dashboard.     |
| `container_last_seen`                       | cAdvisor                 | Last time cAdvisor saw each container.       | Grafana container dashboard.     |

## Label Policy

Telemetry in this demo should keep metrics low-cardinality and dashboard
friendly.

Allowed label shapes:

- stable service/job labels added by Prometheus or the exporter
- bounded operational labels such as `kind`, `operation`, `stage`, or `outcome`
  if new metrics need them

Do not put high-cardinality identifiers into metric labels. These belong in
logs and trace attributes instead.

Forbidden examples:

- transaction hashes
- block header hashes
- UTxO references
- wallet or script addresses
- raw CBOR
- raw error messages
- stack traces
- arbitrary URL paths

## OpenTelemetry And Logs

When `listen --with-monitoring` is used, the node configures an Effect
OpenTelemetry SDK layer with:

- Prometheus metrics exported from the node process,
- OTLP traces exported to Tempo,
- `serviceName` set to `midgard-node`,
- top-level tracing span `midgard`,
- worker/fiber spans such as `block-commitment-fiber`,
  `submit-blocks-fiber`, `sync-user-events-fiber`, and
  `merge-confirmed-state-fiber`.

The Docker stack forwards container logs to Loki through Promtail. Use:

- metrics for rates, gauges, and dashboard trends,
- traces for execution-path debugging,
- logs for detailed identifiers and transaction/block context.

Do not duplicate trace identifiers or transaction identifiers in metric labels.

## Operational Notes

- Docker starts the node with `listen --with-monitoring`, which enables metrics
  and trace export.
- Running `pnpm listen` starts the node without monitoring unless the
  `--with-monitoring` flag is provided.
- The metrics registry is process-local and created once per node process.
- The Grafana dashboard currently expects the Docker-network scrape identity
  `instance="midgard-node:9464"`.
- Tempo is configured with short local retention for demo purposes.

## Out Of Scope

The following are intentionally outside this demo telemetry scope:

- production alert routing and paging policy
- cloud networking or load balancer observability
- long-term metrics, logs, or trace storage
