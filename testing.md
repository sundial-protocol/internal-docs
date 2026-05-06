# Testing

This document summarizes the Midgard node test layers, the main entrypoints,
and when to use each flow.

## Test Layers

The repository uses:

- unit tests for isolated service, worker, or function behavior,
- integration tests for real repository and service wiring inside the Midgard
  node boundary,
- Dockerized end-to-end or system tests for live runtime behavior.

The current seed suite in `demo/midgard-node` runs through `pnpm test`; as the
suite is split out, those tests should map into the command structure below.

## Core Commands

API:

- `npm run test:api:unit`
- `npm run test:api:integration`
- `npm run test:api:e2e`
- `npm run test:api`
- `npm run test:api:all`
- `npm run test:api:cov`

Node internals:

- `npm run test:node:unit`
- `npm run test:node:integration`
- `npm run test:node`
- `npm run test:node:cov`

Aggregates:

- `npm run test:unit`
- `npm run test:integration`
- `npm run test:cov`
- `npm run test:e2e`
- `npm run test:system`
- `npm run test:system:all`
- `npm run tests:all`
- `npm run test:all`

Current demo entrypoints:

- `cd demo/midgard-node && pnpm test`
- `cd demo/midgard-node && pnpm type-check`
- `cd demo/midgard-node && pnpm build`
- `cd demo/midgard-node && docker compose run --rm midgard-node-tests`

## What `tests:all` Runs

`npm run tests:all` runs these commands, in this exact order:

1. `npm run test:unit`
2. `npm run test:integration`
3. `npm run test:cov`
4. `npm run test:e2e`
5. `npm run test:system:all`

Differences between these layers:

1. `test:unit`: Fastest layer, isolated logic tests with mocked collaborators.
   No Docker, minimal external dependencies. Current candidates include
   in-memory MPT behavior and fraud-proof catalogue initialization logic.
2. `test:integration`: Real internal wiring inside the Midgard node boundary.
   Current candidates include Effect SQL repositories, database initialization,
   and LevelDB-backed MPT persistence.
3. `test:cov`: Coverage-focused runs for API and node internals that also
   enforce per-file coverage checks. Slower than plain unit/integration.
4. `test:e2e`: Live Dockerized Midgard node + Postgres contract checks.
   Focused on runtime behavior, startup, CLI commands, and HTTP contracts.
5. `test:system:all`: Cross-service runtime/system flows:
   `test:system:e2e-emulator` (real Midgard node with emulator-backed L1
   boundary), and `test:system:obs` (telemetry/observability smoke flow).

Local default `test:system` intentionally runs only:

- `test:system:e2e-emulator`
- `test:system:obs`

This keeps routine local system checks independent from external L1 provider
credentials.

## Command Selection

Use the narrowest relevant command first:

- API service/controller changes: `npm run test:api:unit`
- repository or SQL mapping changes: `npm run test:api:integration`
- MPT, worker, fiber, or transaction-pipeline changes: `npm run test:node`
- local live API verification: `npm run test:api:e2e`
- cross-service runtime behavior with emulator-backed L1 boundary:
  `npm run test:system:e2e-emulator`
- full local observability smoke coverage: `npm run test:system:obs`

Until the split commands exist, use the current demo commands:

- isolated/current suite verification: `cd demo/midgard-node && pnpm test`
- containerized database parity:
  `cd demo/midgard-node && docker compose run --rm midgard-node-tests`

## Current Seed Suite

`cd demo/midgard-node && pnpm test` sets `NODE_ENV=emulator` and runs Vitest
with the project configuration in `vitest.config.ts`.

The current suite includes:

1. `tests/database.test.ts`: database initialization and repository operation
   checks against the configured PostgreSQL database.
2. `tests/mpt.test.ts`: LevelDB-backed and in-memory MPT behavior.
3. `tests/fraud-proof-catalogue.test.ts`: fraud-proof catalogue root and
   pre-image checks using the always-succeeds contracts.

The Vitest config uses a Node environment, forked workers, `tests/**/*.test.*`
as the include pattern, a 420 second timeout, and `bail: 3`.

## System Flows

### Dockerized API E2E

`npm run test:api:e2e` should:

1. start the Dockerized Midgard node + Postgres stack,
2. wait for health endpoints,
3. run the live E2E spec,
4. tear the stack down.

Preferred local setup: `demo/midgard-node/.env` exists (bootstrap once with
`cp .env.example .env` from `demo/midgard-node`).

This flow should build and run the Docker image path from
`demo/midgard-node/Dockerfile` for test parity with runtime behavior. If the
configured host Postgres or API port is unavailable, the wrapper should
auto-select a free host port and export it for the host-side Vitest process.

### Emulator system flow

`npm run test:system:e2e-emulator` should run:

- real Midgard node,
- Dockerized Postgres,
- emulator-backed L1/provider boundary,
- the Midgard transaction, block commitment, merge, and submission flows that
  can run without external provider credentials.

This is the main cross-service system test without a live external L1 provider.

Preferred local setup: `demo/midgard-node/.env` exists. If the local file is
missing, the wrapper should create a temporary runtime env file from
`.env.example` for that run.

### Observability system flow

`npm run test:system:obs` starts the observability Compose stack
(Midgard node, Postgres, Grafana, Prometheus, Loki, Tempo, Promtail, cAdvisor)
and then runs the dedicated smoke spec.

Preferred local setup: `demo/midgard-node/.env` exists. If the local file is
missing, the wrapper should create a temporary runtime env file from
`.env.example` for that run.

### Dedicated L1 provider runtime flow

`npm run test:system:l1-provider` should run a dedicated runtime wrapper that
starts the Dockerized Midgard node + Postgres stack and applies explicit L1
provider configuration for the mode under test, for example:

- `L1_PROVIDER=Kupmios`
- `L1_PROVIDER=Blockfrost`

Use this for provider-specific smoke coverage that should not be part of the
default local system suite.

## Quality And Security Gates

Broad validation commands:

- `npm run quality:check:all`
- `npm run security:check:all`
- `npm run test:all`

Current demo validation commands:

- `cd demo/midgard-node && pnpm format-check`
- `cd demo/midgard-node && pnpm type-check`
- `cd demo/midgard-node && pnpm build`
- `cd demo/midgard-node && pnpm test`
- `cd demo/midgard-node && docker compose run --rm midgard-node-tests`

## Related Docs

- [Midgard Node README](../../demo/midgard-node/README.md)
- [Midgard Node Data Storage](../../demo/midgard-node/DATA_STORAGE.md)
- [Midgard Node Process Logic](../../demo/midgard-node/PROCESS_LOGIC.md)
