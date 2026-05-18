# Observability

`demo/midgard-node` provides a Docker Compose stack for local runtime +
observability:

- `demo/midgard-node/docker-compose.yaml` (Midgard node + Postgres +
  Prometheus + Loki + Promtail + cAdvisor + Grafana + Tempo)
- `demo/midgard-node/docker-compose.dev.yaml` (Midgard node + Postgres only)

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

Note: the variable name is currently `OLTP_EXPORTER_URL` in the Midgard node
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

Follow Midgard node logs:

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

- Midgard node API: `http://localhost:3000`
- Midgard node metrics: `http://localhost:9464/metrics`
- Prometheus: `http://localhost:9090`
- Loki: `http://localhost:3100`
- cAdvisor: `http://localhost:8080`
- Grafana: `http://localhost:3001`
- Tempo: `http://localhost:3200`
- Tempo OTLP HTTP receiver: `http://localhost:4318`

The Midgard node exposes routes such as:

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

There is no dedicated health endpoint in this demo stack.

## Persistence

The full Compose stack persists the following Docker volumes:

- `postgres-data`
- `prometheus-data`
- `grafana-data`
- `tempo-data`

The Midgard node service also bind-mounts `./db` into the container at
`/app/db`.

## Smoke Checks

1. Confirm containers are running:

```bash
cd demo/midgard-node
docker compose ps
```

2. Confirm the Midgard node API is reachable:

```bash
curl -fsS http://localhost:3000/stateQueue
```

3. Confirm Prometheus can scrape targets:

```bash
curl -fsS 'http://localhost:9090/api/v1/targets?state=active'
```

Prometheus should include scrape jobs for:

- `prometheus`
- `midgard_nodes`
- `cadvisor`
- `tempo`

4. Open Grafana at `http://localhost:3001` and verify the provisioned data
   sources:

- `prometheus`
- `Loki`
- `Tempo`

5. Generate traffic by calling Midgard node endpoints and check:

- logs appear in Grafana Explore (Loki),
- metrics appear in Prometheus/Grafana,
- traces appear in Grafana Explore (Tempo).

## Troubleshooting

- `docker compose up` fails with missing env values:
  - verify `demo/midgard-node/.env` exists and is populated.
- `midgard_nodes` metrics target is down in Prometheus:
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
    Midgard node container.
- Grafana opens but dashboards or data sources are missing:
  - verify the provisioning mounts under `demo/midgard-node/grafana/` exist and
    the `grafana` service started after Prometheus, Loki, and Tempo.
- Local port conflicts:
  - check for existing services on ports `3000`, `3001`, `5433`, `8080`,
    `9090`, `3100`, `3200`, `4317`, or `4318`, then stop them or adjust the
    Compose port mappings.
