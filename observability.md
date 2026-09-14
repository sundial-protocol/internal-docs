# Observability

[`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node)
provides a Docker Compose stack for local runtime + observability:

- [`sundial-node/docker-compose.yaml`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/docker-compose.yaml)
  (Sundial node + Postgres + Prometheus + Loki + Promtail + cAdvisor +
  Grafana + Tempo)
- [`sundial-node/docker-compose.dev.yaml`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/docker-compose.dev.yaml)
  (Sundial node + Postgres only)

## Prerequisites

Create the local env file:

```bash
cd demo/midgard-node
cp .env.example .env
```

Fill the L1/provider values needed for the selected `L1_PROVIDER`. When running
with `L1_PROVIDER=Kupmios`, `L1_BLOCKFROST_API_URL` and `L1_BLOCKFROST_KEY` can
stay empty.

The full Compose stack starts `midgard-node` with `listen --with-monitoring`,
which enables the Prometheus exporter and OTLP trace exporter. The relevant
defaults in `.env.example` are:

- `PORT=3000`
- `PROM_METRICS_PORT=9464`
- `OLTP_EXPORTER_URL=http://tempo:4318/v1/traces`

Note: the variable name is currently `OLTP_EXPORTER_URL` in the Sundial node
code and env file.

## Start and Stop

Full local runtime + observability stack:

```bash
cd demo/midgard-node
docker compose up -d --build
docker compose down
```

Development stack without monitoring services:

```bash
cd demo/midgard-node
docker compose -f docker-compose.dev.yaml up -d --build
docker compose -f docker-compose.dev.yaml down
```

Follow Sundial node logs:

```bash
cd demo/midgard-node
docker logs -f midgard-midgard-node-1
```

Remove the full stack and its Docker volumes:

```bash
cd demo/midgard-node
docker compose down -v
```

Run the Docker-backed test service:

```bash
cd demo/midgard-node
docker compose run --rm midgard-node-tests
```

## Exposed Local Endpoints

- Sundial node API: `http://localhost:3000`
- Sundial node metrics: `http://localhost:9464/metrics`
- Prometheus: `http://localhost:9090`
- Loki: `http://localhost:3100`
- cAdvisor: `http://localhost:8080`
- Grafana: `http://localhost:3001`
- Tempo: `http://localhost:3200`
- Tempo OTLP HTTP receiver: `http://localhost:4318`

The Sundial node exposes routes such as:

- `GET /tx`
- `GET /txs`
- `GET /utxos`
- `GET /block`
- `GET /init`
- `GET /commit`
- `GET /merge`
- `GET /reset`
- `GET /stateQueue`
- `POST /submit`
- `GET /health/live`
- `GET /health/ready`

`GET /health/live` and `GET /health/ready` are the liveness / readiness probes
used by the cloud deployment's target group; `/health/ready` returning `200` is
the cutover verification signal.

## Persistence

The full Compose stack persists the following Docker volumes:

- `postgres-data`
- `prometheus-data`
- `grafana-data`
- `tempo-data`

The Sundial node service also bind-mounts `./db` into the container at
`/app/db`.

### Retention

Metrics and logs are retained long enough for retrospective reliability
reporting over a closed calendar window (see
[`telemetry.md`](telemetry.md#retention--persistence) and
[`reliability-reporting.md`](reliability-reporting.md)):

- Prometheus TSDB: `PROMETHEUS_RETENTION_TIME` (default `120d`) with a
  `PROMETHEUS_RETENTION_SIZE` cap (default `40GB`), set on the `prometheus`
  service `command` in `docker-compose.yaml`. Cloud: `prometheus_retention`
  / `prometheus_retention_size` in `envs/testnet.tfvars`.
- Loki: `retention_period: 120d` with compactor retention enabled
  (`loki-config.yaml`). Cloud: `loki_retention_days`.
- Tempo: `block_retention: 72h` (`tempo.yaml`); no Tempo in the cloud.

In the cloud deployment, Prometheus and Loki data are on EFS access points
(`infra/aws/terraform/platform/efs.tf`), not ECS `host_path` volumes, so
retention survives task rescheduling.

### Prometheus snapshot

The admin API is enabled locally (`--web.enable-admin-api`) and in the cloud
(kept on the private subnet). To freeze a point-in-time copy of the TSDB — for
an evidence bundle or an offline investigation:

```bash
# Trigger a snapshot (writes under <tsdb>/snapshots/<name>)
curl -s -XPOST http://localhost:9090/api/v1/admin/tsdb/snapshot

# Local: the snapshot dir is inside the prometheus-data volume
docker compose exec prometheus ls /prometheus/snapshots
```

## Smoke Checks

1. Confirm containers are running:

```bash
cd demo/midgard-node
docker compose ps
```

2. Confirm the Sundial node API is reachable:

```bash
curl -fsS http://localhost:3000/stateQueue
```

3. Confirm Prometheus can scrape targets:

```bash
curl -fsS 'http://localhost:9090/api/v1/targets?state=active'
```

Prometheus should include scrape jobs for:

- `prometheus`
- `sundial_nodes`
- `cadvisor`
- `tempo`

`curl -fsS http://localhost:9090/api/v1/rules` should list the `slo_*` groups
loaded from `rules/` — regenerate them with `pnpm run slo:check` if missing.

4. Open Grafana at `http://localhost:3001` and verify the provisioned data
   sources:

- `prometheus`
- `Loki`
- `Tempo`

and the provisioned dashboards under the `Services` folder:

- the main ops dashboard (`dashboard.json`)
- `reliability.json` — the SLO / reliability dashboard (SLI success ratios vs
  objectives, latency quantiles, error-budget-remaining and burn rate,
  availability, restarts, block cadence)

5. Generate traffic by calling Sundial node endpoints and check:

- logs appear in Grafana Explore (Loki),
- metrics appear in Prometheus/Grafana,
- traces appear in Grafana Explore (Tempo).

## Troubleshooting

- `docker compose up` fails with missing env values:
  - verify `demo/midgard-node/.env` exists and is populated.
- `sundial_nodes` metrics target is down in Prometheus:
  - confirm the full `docker-compose.yaml` stack is running, not
    `docker-compose.dev.yaml`.
  - confirm `PROM_METRICS_PORT=9464` and that the node starts with
    `listen --with-monitoring`.
- No logs in Loki:
  - verify Promtail is running and can read Docker logs through
    `/var/run/docker.sock` and `/var/lib/docker/containers`.
  - confirm the `midgard-node` service has the `logging=promtail` label.
- Loki fails to start repeatedly:
  - run `docker compose down -v`, then retry `docker compose up -d --build`.
- No traces in Tempo:
  - verify `OLTP_EXPORTER_URL=http://tempo:4318/v1/traces` is active in the
    Sundial node container.
- Grafana opens but dashboards or data sources are missing:
  - verify the provisioning mounts under
    [`sundial-node/grafana`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node/grafana)
    exist and the `grafana` service started after Prometheus, Loki, and
    Tempo.
- Local port conflicts:
  - check for existing services on ports `3000`, `3001`, `5433`, `8080`,
    `9090`, `3100`, `3200`, `4317`, or `4318`, then stop them or adjust the
    Compose port mappings.
