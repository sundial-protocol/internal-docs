# Midgard Environment

This document describes the environment and local configuration used by the
Midgard demo under `demo/`.

## Env Files

The node template lives at:

- `demo/midgard-node/.env.example`

The local runtime file is:

- `demo/midgard-node/.env`

Bootstrap it with:

```bash
cd demo/midgard-node
cp .env.example .env
```

`dotenv.config()` is called by `demo/midgard-node/src/index.ts`, so local
non-Docker runs read `.env` from the `midgard-node` working directory.

Docker Compose also uses:

```yaml
env_file:
  - .env
```

and uses some of the same variables for Compose substitution, especially the
PostgreSQL settings.

## Runtime Contract

Configuration is parsed by `demo/midgard-node/src/services/config.ts` using
Effect Config.

Important behavior:

- `L1_PROVIDER` must be either `Kupmios` or `Blockfrost`.
- `NETWORK` must be one of `Mainnet`, `Preprod`, `Preview`, or `Custom`.
- `PORT`, timing intervals, metrics port, Postgres settings, MPT paths, and
  OTLP URL have code defaults.
- L1 provider fields and operator seed phrases are read as strings and should
  be present in `.env`.
- The example file keeps Blockfrost fields empty for `Kupmios` mode, but the
  variables still exist.
- Testnet genesis seed phrases are required by the current config loader; they
  are used to derive hard-coded demo genesis UTxO addresses when
  `NETWORK !== Mainnet`.

## Node Env

These variables are defined by `demo/midgard-node/.env.example`.

| Variable                                       | Required by config | Default in code                 | Purpose                                                                                           |
| ---------------------------------------------- | ------------------ | ------------------------------- | ------------------------------------------------------------------------------------------------- |
| `COMPOSE_PROJECT_NAME`                         | Compose only       | n/a                             | Docker Compose project name, set to `midgard` in the example.                                     |
| `L1_PROVIDER`                                  | Yes                | none                            | Cardano provider mode: `Kupmios` or `Blockfrost`.                                                 |
| `L1_BLOCKFROST_API_URL`                        | Present as string  | none                            | Blockfrost API URL used only when `L1_PROVIDER=Blockfrost`.                                       |
| `L1_BLOCKFROST_KEY`                            | Present as string  | none                            | Blockfrost API key used only when `L1_PROVIDER=Blockfrost`.                                       |
| `L1_OGMIOS_KEY`                                | Present as string  | none                            | Ogmios endpoint/value passed to Lucid `Kupmios`.                                                  |
| `L1_KUPO_KEY`                                  | Present as string  | none                            | Kupo endpoint/value passed to Lucid `Kupmios`.                                                    |
| `L1_OPERATOR_SEED_PHRASE`                      | Yes                | none                            | Operator main wallet seed phrase.                                                                 |
| `L1_OPERATOR_SEED_PHRASE_FOR_BLOCK_COMMITMENT` | Yes                | none                            | Wallet seed used for block commitment transactions.                                               |
| `L1_OPERATOR_SEED_PHRASE_FOR_MERGE_TX`         | Yes                | none                            | Wallet seed used for state-queue merge transactions.                                              |
| `NETWORK`                                      | Yes                | none                            | Lucid network: `Mainnet`, `Preprod`, `Preview`, or `Custom`.                                      |
| `PORT`                                         | No                 | `3000`                          | Midgard node HTTP RPC port.                                                                       |
| `WAIT_BETWEEN_BLOCK_COMMITMENTS`               | No                 | `1000`                          | Delay in milliseconds between block commitment fiber runs.                                        |
| `WAIT_BETWEEN_BLOCK_SUBMISSIONS`               | No                 | `10000`                         | Delay in milliseconds between block submission fiber runs.                                        |
| `WAIT_BETWEEN_USER_EVENT_FETCHES`              | No                 | `10000`                         | Delay in milliseconds between L1 user-event sync runs.                                            |
| `WAIT_BETWEEN_MERGE_TXS`                       | No                 | `10000`                         | Delay in milliseconds between merge fiber runs.                                                   |
| `PROM_METRICS_PORT`                            | No                 | `9464`                          | Prometheus exporter port when monitoring is enabled.                                              |
| `OLTP_EXPORTER_URL`                            | No                 | `http://0.0.0.0:4318/v1/traces` | OTLP HTTP trace exporter URL. The variable name is currently spelled `OLTP`, not `OTLP`, in code. |
| `POSTGRES_USER`                                | No                 | `postgres`                      | PostgreSQL username. Also used by Docker Compose.                                                 |
| `POSTGRES_PASSWORD`                            | No                 | `postgres`                      | PostgreSQL password. Also used by Docker Compose.                                                 |
| `POSTGRES_DB`                                  | No                 | `midgard`                       | PostgreSQL database name. Also used by Docker Compose.                                            |
| `POSTGRES_HOST`                                | No                 | `postgres`                      | PostgreSQL host. Use `postgres` in Docker and commonly `localhost` outside Docker.                |
| `LEDGER_MPT_DB_PATH`                           | No                 | `midgard-ledger-mpt-db`         | Filesystem path for the ledger MPT LevelDB store.                                                 |
| `MEMPOOL_MPT_DB_PATH`                          | No                 | `midgard-mempool-mpt-db`        | Filesystem path for the mempool MPT LevelDB store.                                                |
| `TESTNET_GENESIS_WALLET_SEED_PHRASE_A`         | Yes                | none                            | Demo seed used to derive testnet genesis UTxO addresses.                                          |
| `TESTNET_GENESIS_WALLET_SEED_PHRASE_B`         | Yes                | none                            | Demo seed used to derive testnet genesis UTxO addresses.                                          |
| `TESTNET_GENESIS_WALLET_SEED_PHRASE_C`         | Yes                | none                            | Demo seed used to derive testnet genesis UTxO addresses.                                          |

## Provider Modes

### Kupmios

Use:

```env
L1_PROVIDER=Kupmios
L1_KUPO_KEY=<kupo-url-or-config-value>
L1_OGMIOS_KEY=<ogmios-url-or-config-value>
```

The node creates Lucid with:

```ts
new LE.Kupmios(nodeConfig.L1_KUPO_KEY, nodeConfig.L1_OGMIOS_KEY);
```

The Blockfrost variables may be empty in this mode, as shown in
`.env.example`, but they should still be present.

### Blockfrost

Use:

```env
L1_PROVIDER=Blockfrost
L1_BLOCKFROST_API_URL=<blockfrost-api-url>
L1_BLOCKFROST_KEY=<blockfrost-api-key>
```

The node creates Lucid with:

```ts
new LE.Blockfrost(
  nodeConfig.L1_BLOCKFROST_API_URL,
  nodeConfig.L1_BLOCKFROST_KEY,
);
```

The Kupmios variables are still read by config and should be present in `.env`.

## PostgreSQL

The node connects through `@effect/sql-pg` with:

- host: `POSTGRES_HOST`
- username: `POSTGRES_USER`
- password: `POSTGRES_PASSWORD`
- database: `POSTGRES_DB`
- `maxConnections=20`
- `idleTimeout=5 minutes`
- `connectTimeout=2 seconds`

Docker Compose publishes PostgreSQL on host port `5433` and container port
`5432`.

For non-Docker local runs, the README recommends:

```env
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=midgard
POSTGRES_HOST=localhost
LEDGER_MPT_DB_PATH=midgard-ledger-mpt-db
MEMPOOL_MPT_DB_PATH=midgard-mempool-mpt-db
```

## MPT Storage

The node uses filesystem-backed stores for:

- ledger MPT: `LEDGER_MPT_DB_PATH`
- mempool MPT: `MEMPOOL_MPT_DB_PATH`

In Docker Compose, `./db` is mounted as `/app/db`, but the default paths are
relative unless changed in `.env`.

`GET /reset` deletes these MPT stores through the reset program.

## Monitoring Env

Monitoring is enabled by starting the node with `--with-monitoring`.

The full Docker Compose runtime uses:

- `PROM_METRICS_PORT` for the Prometheus exporter,
- `OLTP_EXPORTER_URL` for traces sent to Tempo,
- Prometheus on host port `9090`,
- Grafana on host port `3001`,
- Loki on host port `3100`,
- cAdvisor on host port `8080`,
- Tempo on host ports `14268`, `3200`, `9095`, `4317`, `4318`, and `9411`.

The development Compose file starts only:

- `midgard-node`,
- `postgres`.

## Manager Configuration

`demo/midgard-manager` does not use `.env` for its main settings. It uses JSON
configuration files under `demo/midgard-manager/config`.

The checked-in default is:

```json
{
  "node": {
    "endpoint": "http://localhost:3000"
  },
  "generator": {
    "enabled": false,
    "batchSize": 10,
    "intervalMs": 1000
  }
}
```

The CLI schema also supports:

- `generator.maxConcurrent`,
- `logging.level`,
- `logging.format`.

The schema defaults in `packages/cli/src/config/schema.ts` are:

```json
{
  "node": {
    "endpoint": "http://localhost:3000"
  },
  "generator": {
    "enabled": false,
    "maxConcurrent": 10,
    "batchSize": 100,
    "intervalMs": 1000
  },
  "logging": {
    "level": "info",
    "format": "pretty"
  }
}
```

The transaction generator also accepts CLI options such as:

- `--endpoint`
- `--type`
- `--ratio`
- `--batch-size`
- `--interval`
- `--concurrency`
- `--test-wallet`
- `--private-key`

## Build And Test Notes

The demo workspace uses pnpm. Common commands:

```bash
cd demo
pnpm install
pnpm build
pnpm test
```

Node-specific commands:

```bash
cd demo/midgard-node
pnpm build
pnpm listen
pnpm test
```

SDK packing flow used by `midgard-node`:

```bash
cd demo/midgard-sdk
pnpm install
pnpm repack
```

The node package depends on the packed SDK tarball:

```json
"@al-ft/midgard-sdk": "file:../midgard-sdk/al-ft-midgard-sdk-0.1.0.tgz"
```

## Related Docs

- [Architecture](./architecture.md)
- [API](./api.md)
