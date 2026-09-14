# Telemetry

[`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node)
exposes scrapeable Prometheus metrics from the Sundial node process when it
is started with monitoring enabled.

This document focuses on application metrics, tracing, and telemetry semantics
for the node. It also ships a local/container observability stack
through Docker Compose: Prometheus, Grafana, Loki, Promtail, Tempo, and
cAdvisor. The stack configuration lives under
[`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node).

## Endpoints

- Node API: `http://localhost:3000` by default, controlled by `PORT`.
- Node metrics: `GET /metrics` on the OpenTelemetry Prometheus exporter port.
- Node health: `GET /health/live` and `GET /health/ready` on the API port.
- Prometheus UI: `http://localhost:9090`
- Grafana UI: `http://localhost:3001`
- Loki: `http://localhost:3100`
- Tempo: `http://localhost:3200`
- cAdvisor: `http://localhost:8080`

Example local scrape target:

- Sundial node: `http://localhost:9464/metrics` when
  `PROM_METRICS_PORT=9464`

Prometheus scrapes the node through the `sundial_nodes` job. Locally the job
targets `node:9464` (and `node-api:9464` / `node-tx-processor:9464` /
`node-sequencer:9464` in the split-role profile), each carrying a `role` label.

## Implementation Locations

Node telemetry setup lives in:

- [`sundial-node/src/commands/listen.ts`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/src/commands/listen.ts)
- [`sundial-node/src/services/config.ts`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/src/services/config.ts)

Application metrics live in:

- [`sundial-node/src/commands/listen.ts`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/src/commands/listen.ts)
- [`sundial-node/src/commands/http-metrics.ts`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/src/commands/http-metrics.ts)
- [`sundial-node/src/fibers/block-commitment.ts`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/src/fibers/block-commitment.ts)
- [`sundial-node/src/fibers/block-submission.ts`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/src/fibers/block-submission.ts)
- [`sundial-node/src/fibers/monitor-mempool.ts`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/src/fibers/monitor-mempool.ts)
- [`sundial-node/src/fibers/tx-queue-processor.ts`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/src/fibers/tx-queue-processor.ts)
- [`sundial-node/src/transactions/state-queue/merge-to-confirmed-state.ts`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/src/transactions/state-queue/merge-to-confirmed-state.ts)

SLO/SLI definitions and the generated Prometheus recording + alert rules live in:

- [`sundial-node/slo/slo.json`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/slo/slo.json) — single source of truth
- [`sundial-node/slo/gen-rules.mjs`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/slo/gen-rules.mjs) — generator (`pnpm run slo:gen` / `slo:check`)
- [`sundial-node/rules/`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node/rules) — generated `slo-recording.rules.yml` + `slo-alerts.rules.yml`

Observability stack configuration lives in:

- [`sundial-node/docker-compose.yaml`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/docker-compose.yaml)
- [`sundial-node/prometheus.yml`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/prometheus.yml)
- [`sundial-node/grafana`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node/grafana)
- [`sundial-node/loki-config.yaml`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/loki-config.yaml)
- [`sundial-node/promtail-config.yaml`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/promtail-config.yaml)
- [`sundial-node/tempo.yaml`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/tempo.yaml)

## Metrics Endpoint Env

- `PROM_METRICS_PORT`: Prometheus exporter port; defaults to `9464`.
- `OLTP_EXPORTER_URL`: OTLP HTTP trace endpoint; defaults to
  `http://0.0.0.0:4318/v1/traces`.

The metrics endpoint is separate from the node API port.

## Application Metrics

Counters are exported by the OpenTelemetry Prometheus exporter with a `_total`
suffix (the code registers them without it, e.g. `commit_block_count` →
`commit_block_count_total` when scraped). Metric definitions in the source are
the authoritative catalogue; the tables below cover the series the dashboards
and the SLO rules depend on.

### Ingress (`POST /submit`)

| Metric                                     | Type      | Labels                          | Meaning                                                             |
| ------------------------------------------ | --------- | ------------------------------- | ------------------------------------------------------------------ |
| `tx_submissions_enqueued_total`            | Counter   | none                            | L2 tx submissions that passed hex validation and were enqueued.    |
| `tx_submissions_rejected_total`            | Counter   | none                            | L2 tx submissions rejected at ingress (malformed / non-hex CBOR).  |
| `tx_submissions_mempool_accepted_total`    | Counter   | none                            | Enqueued submissions the processor accepted into `MempoolDB`.      |
| `http_server_requests_total`               | Counter   | `route`, `method`, `status_class` | HTTP requests handled, by bounded route enum and status class.   |
| `http_server_request_duration_seconds`     | Histogram | `route`, `method`, `status_class` | HTTP request latency; source for the `submit_ingress_latency` SLI. |

`route` is a fixed enum (`submit`, `health/live`, `health/ready`, the other
known API paths, or `other`). `POST /submit` is intercepted before the Effect
router and reported from
[`raw-submit-interceptor.ts`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/src/services/raw-submit-interceptor.ts);
every other route is timed by a router middleware in `http-metrics.ts`.

### Block pipeline

| Metric                                       | Type      | Labels | Meaning                                                                        |
| -------------------------------------------- | --------- | ------ | ----------------------------------------------------------------------------- |
| `mempool_tx_count`                           | Gauge     | none   | Current number of transactions in `MempoolDB`.                                |
| `commit_block_count_total`                   | Counter   | none   | Committed Sundial blocks.                                                     |
| `commit_block_tx_count_total`                | Counter   | none   | Transaction requests included in committed blocks.                            |
| `commit_block_commitment_failures_total`     | Counter   | none   | Commitment-worker failures (timeout, crash, SDK/CML error).                   |
| `commit_block_txs_per_block`                 | Gauge     | none   | Transaction-request count in the latest committed block.                      |
| `commit_block_events_size_bytes`             | Gauge     | none   | Event size of the latest committed block.                                     |
| `commit_block_duration_seconds`              | Histogram | none   | Block-commitment worker duration (success or failure).                        |
| `commitment_window_age_seconds`             | Gauge     | none   | Age of the commitment window for the last committed block (last value).       |
| `tx_ingress_to_commit_duration_seconds`      | Histogram | none   | Commitment-window span per committed block — inclusion-latency distribution (upper bound on mempool-accepted → committed). Source for `inclusion_latency`. |
| `submit_block_count_total`                   | Counter   | none   | Blocks submitted to L1.                                                       |
| `submit_block_failures_total`                | Counter   | none   | L1 submission failures.                                                       |
| `submit_block_submit_duration_seconds`       | Histogram | none   | L1 submit duration.                                                           |
| `l1_commitment_precheck_confirmed_total`     | Counter   | none   | Blocks whose L1 commitment was already on-chain at the pre-submit check (idempotent recovery after an earlier timed-out attempt). |
| `l1_commitment_idempotent_recovered_total`   | Counter   | none   | Submissions where L1 reported a conflict but a follow-up inclusion check confirmed the commitment had landed. |
| `l1_commitment_fees_lovelace_total`          | Counter   | none   | Cumulative L1 commitment fees paid, in lovelace.                              |
| `merge_block_count_total`                    | Counter   | none   | Blocks merged into confirmed state.                                           |
| `merge_block_failures_total`                 | Counter   | none   | Merge-worker failures.                                                        |
| `unsubmitted_block_backlog`                  | Gauge     | none   | Committed-but-not-yet-submitted block backlog.                               |

### Durable ingress stream

`tx_stream_depth`, `tx_stream_depth_peak`, `tx_stream_pending`,
`tx_stream_consumer_lag`, `tx_stream_ack_total`, `tx_stream_fail_total`,
`tx_stream_retry_total`, `tx_stream_dead_letter_total` — Redis Streams ingress
queue depth / lag / ack / failure / retry / dead-letter signals
([`tx-queue-processor.ts`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/src/fibers/tx-queue-processor.ts)).

### Build / process identity

| Metric                            | Type  | Labels                                             | Meaning                                                            |
| --------------------------------- | ----- | ------------------------------------------------- | ---------------------------------------------------------------- |
| `midgard_node_build_info`         | Gauge | `version`, `commit`, `l1_provider`, `network`, `role` | Always `1`. Provenance anchor; the reliability report attributes a window to a specific build and detects redeploys. |
| `midgard_node_start_time_seconds` | Gauge | none                                              | Unix seconds at which the node process started; source for uptime / restart detection. |

`commit` comes from the `MIDGARD_NODE_COMMIT` env var, injected at image build
via `docker build --build-arg MIDGARD_NODE_COMMIT=$(git rev-parse --short HEAD)`
(see [`Dockerfile`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/Dockerfile)).
It is `unknown` for a locally-run node that was not built through that path.

`midgard_node_build_info` and the HTTP metrics are emitted only with their full
label set (no untagged `…{} 0` series); `build_info` appears at startup, the
HTTP series within seconds of the first request.

## Infrastructure Metrics

The local Docker Compose stack also scrapes metrics that do not originate inside
the Sundial node code:

| Metric                                   | Source                   | Meaning                                      | Default usage                    |
| ---------------------------------------- | ------------------------ | --------------------------------------------- | --------------------------------- |
| `up{job="sundial_nodes"}`                | Prometheus target health | Scrape health for the Sundial node exporter. | Availability SLI and scrape context. |
| `container_memory_usage_bytes`           | cAdvisor                 | Container memory usage.                      | Grafana container dashboard.     |
| `container_cpu_user_seconds_total`       | cAdvisor                 | Container CPU usage counter.                 | Grafana container dashboard.     |
| `container_network_receive_bytes_total`  | cAdvisor                 | Container receive traffic counter.           | Grafana container dashboard.     |
| `container_network_transmit_bytes_total` | cAdvisor                 | Container transmit traffic counter.          | Grafana container dashboard.     |
| `container_last_seen`                    | cAdvisor                 | Last time cAdvisor saw each container.       | Grafana container dashboard.     |

## SLO Recording & Alert Rules

`slo/slo.json` defines seven SLIs — submit-ingress availability and latency,
end-to-end tx success, L1 commitment success, merge success, inclusion latency,
and node scrape availability — each with its PromQL, objective and windows.
`slo/gen-rules.mjs` compiles it into `rules/slo-recording.rules.yml` (ratio /
latency / availability rollups over 5m–30d, burn rates, and
error-budget-remaining) and `rules/slo-alerts.rules.yml` (multi-window
burn-rate, latency and availability alerts). Regenerate and validate with:

```bash
cd demo/midgard-node
pnpm run slo:check   # regenerate + promtool check/test + fail on drift
```

Recorded series follow the pattern `slo:<sli-id>:ratio_rate<window>`,
`slo:<sli-id>:{p50,p95,p99}_<window>`, `slo:<sli-id>:burn_rate<window>` and
`slo:<sli-id>:error_budget_remaining`. The
local Prometheus loads `rules/*.yml` via `rule_files` in `prometheus.yml`; the
cloud Prometheus embeds the same two committed files at `tofu apply` time. The
retrospective reliability report reads objectives from the same `slo.json`, so
report, recording rules and alerts cannot drift apart.

## Retention & Persistence

Metrics and logs are retained long enough to support retrospective reliability
reporting over a closed calendar window (see
[`reliability-reporting.md`](reliability-reporting.md)).

| Store      | Local (`docker-compose.yaml` / config)                              | Cloud (`infra/aws/terraform/platform`)                    |
| ---------- | ------------------------------------------------------------------ | -------------------------------------------------------- |
| Prometheus | `--storage.tsdb.retention.time` `${PROMETHEUS_RETENTION_TIME:-120d}`, size cap `${PROMETHEUS_RETENTION_SIZE:-40GB}` | `prometheus_retention = "120d"`, `prometheus_retention_size = "40GB"` |
| Loki       | `limits_config.retention_period: 120d` + compactor `retention_enabled` (`loki-config.yaml`) | `loki_retention_days = 120` (rendered into the Loki config, compactor retention enabled) |
| Tempo      | `compactor.compaction.block_retention: 72h` (`tempo.yaml`)          | no Tempo in cloud                                        |

In the cloud, Prometheus and Loki data live on **EFS access points**
(`efs.tf`), not ECS `host_path` volumes, so retention survives task
rescheduling. The Prometheus admin API (`--web.enable-admin-api`) is enabled on
both so a point-in-time TSDB snapshot can be taken for an evidence bundle; keep
that endpoint on the private subnet.

Switching the cloud volumes to EFS forces new task definitions — history starts
fresh from the `tofu apply` that introduces it; the previous ~3-day `host_path`
data is not migrated.

## Label Policy

Telemetry should keep metrics low-cardinality and dashboard
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
- Grafana provisions every dashboard under `grafana/` (`dashboard.json` plus
  `reliability.json`, the SLO / reliability dashboard) from one file provider.
- The dashboards key on the `sundial_nodes` job and the local scrape identity
  `instance="node:9464"`.
- Trace retention is 72h locally (`tempo.yaml`); there is no Tempo in the cloud
  deployment.

## Out Of Scope

The following are intentionally outside this testnet telemetry scope:

- cloud networking or load balancer observability
- distributed tracing storage in the cloud deployment
