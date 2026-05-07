# Midgard Demo Architecture

This document describes the Midgard project currently vendored under `demo/`.
It is based on the active code in `demo/midgard-node`,
`demo/midgard-sdk`, `demo/midgard-ts`, and `demo/midgard-manager`.

## Context

Midgard is a Cardano L2 demo implementation. Sundial uses a fork
of this Midgard stack as its Cardano L2 substrate to support Bitcoin yield
product, whichs remain a separate Sundial protocol layer.

Midgard has the following stack:

- `midgard-node` is an Effect-based HTTP node and background worker process.
- Cardano L1 is accessed through Lucid Evolution using either Kupmios or
  Blockfrost.
- PostgreSQL stores the node's relational projection.
- LevelDB-backed Merkle Patricia Trie stores keep the ledger and mempool MPTs.
- The node exposes RPC-style HTTP endpoints, not a NestJS REST API.

The node is still demo/MVP code. Some manager/client utilities reference routes
that are not registered by the node today; those mismatches are documented in
[API](./api.md).

## Project Layout

The active Midgard folders are:

| Folder                 | Purpose                                                                                                                                |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `demo/midgard-node`    | Runtime node, HTTP RPC surface, PostgreSQL access, background fibers, block commitment/submission, L1 user-event sync, monitoring.     |
| `demo/midgard-sdk`     | Off-chain SDK for building operator, watcher, initialization, state-queue, user-event, settlement, and fraud-proof transactions.       |
| `demo/midgard-ts`      | Pure TypeScript binary codec and Cardano type round-trip helpers for Midgard blocks, transactions, outputs, deposits, and withdrawals. |
| `demo/midgard-manager` | CLI and transaction generator for configuring a node endpoint and generating/submitting test L2 transactions.                          |
| `demo/schemes`         | Excalidraw protocol diagrams and an older project-structure sketch.                                                                    |

## Runtime Services

The runnable service is:

- `midgard-node`: HTTP server plus background fibers.

Local Docker Compose also starts:

- PostgreSQL,
- Prometheus,
- Grafana,
- Loki,
- Promtail,
- Tempo,
- cAdvisor.

External dependencies are:

- Cardano L1 provider through Lucid Evolution:
  - `Kupmios` mode uses Kupo plus Ogmios endpoints,
  - `Blockfrost` mode uses a Blockfrost API URL and key.
- Operator wallets supplied as seed phrases.

## Current Stack

- TypeScript, ESM
- Node.js 18+ for the demo workspace
- pnpm 10 at `demo/` level, with older pnpm metadata in
  `demo/midgard-manager`
- Effect, `@effect/platform`, `@effect/sql-pg`, `@effect/opentelemetry`
- Lucid Evolution for Cardano access
- PostgreSQL 15
- LevelDB / memory-level for MPT storage
- `@ethereumjs/mpt`
- `@aiken-lang/merkle-patricia-forestry`
- Aiken-generated Plutus V3 blueprints under
  `demo/midgard-node/blueprints/always-succeeds`
- Vitest for node/sdk tests; Jest remains in `midgard-ts`
- Docker Compose for local runtime and monitoring

## Node Boot Flow

`demo/midgard-node/src/index.ts` loads `.env`, registers the `listen` command,
and runs `runNode(withMonitoring)` with these Effect services:

- `NodeConfig`
- `Database`
- `AlwaysSucceedsContract`
- `Lucid`
- `Globals`

`runNode` performs the following startup work:

1. Reads configuration from environment.
2. Creates an in-memory transaction queue.
3. Initializes PostgreSQL tables through `DBInitialization.program`.
4. Runs `Genesis.program`.
5. Starts the HTTP server on `PORT`.
6. Starts background fibers for block commitment, block submission, L1
   user-event sync, merge, optional mempool monitoring, and transaction queue
   processing.
7. When `--with-monitoring` is supplied, starts the Prometheus exporter on
   `PROM_METRICS_PORT` and exports traces to `OLTP_EXPORTER_URL`.

## Core Components

### HTTP RPC server

The server lives in `demo/midgard-node/src/commands/listen.ts`.

It exposes unversioned RPC-style routes such as:

- transaction lookup,
- address UTxO lookup,
- address history lookup,
- block transaction lookup,
- transaction submission,
- initialization, reset, commit, merge, and state-queue debug operations.

For route details, see [API](./api.md).

### Transaction queue processor

`txQueueProcessorFiber` drains the in-memory queue populated by
`POST /submit`.

For each queued CBOR hex string it:

1. deserializes the Cardano transaction,
2. computes its transaction hash,
3. extracts spent inputs,
4. extracts produced outputs,
5. inserts the processed transaction into `MempoolDB`.

`MempoolDB.insertMultiple` is responsible for updating the mempool-side
projection, including `MempoolLedgerDB` and address-history effects through the
database layer.

### L1 user-event sync

`syncUserEventsFiber` periodically fetches Midgard user-event UTxOs from
Cardano L1:

- deposits,
- transaction orders,
- withdrawals.

It uses the validator addresses and policy IDs from `AlwaysSucceedsContract`
and stores discovered events in:

- `DepositsDB`,
- `TxOrdersDB`,
- `WithdrawalsDB`.

The fetch window is tracked in memory by
`Globals.LATEST_USER_EVENTS_FETCH_TIME`.

### Block commitment

`blockCommitmentFiber` periodically starts a worker thread implemented under
`demo/midgard-node/src/workers/block-commitment.ts`.

The commitment path:

1. reads the latest block and the next event interval,
2. collects user events and mempool transactions in that interval,
3. applies withdrawals, transaction orders, transaction requests, and deposits
   to the ledger model,
4. computes ledger, deposit, withdrawal, and transaction roots,
5. builds and signs a Cardano transaction that appends the new block to the
   Midgard state queue,
6. stores the block in `BlocksDB` with `Unsubmitted` status.

The worker also updates metrics for committed block count, transactions, event
counts, and total event size.

### Block submission

`blockSubmissionFiber` looks up the earliest unsubmitted block from `BlocksDB`
and submits its previously built L1 transaction.

After a successful submit it:

- processes events in the block interval,
- updates `LatestLedgerDB`,
- moves mempool transactions into `ImmutableDB`,
- inserts block-to-transaction mappings into `BlocksTxsDB`,
- upserts address-history rows,
- marks the block as `Submitted`.

### Merge

`mergeFiber` periodically builds and submits a merge transaction for the
confirmed state queue using the state-queue validator from the SDK.

The manual `GET /merge` endpoint runs the same action.

### Reset

`Reset.program` is development-oriented. It:

- spends and burns Midgard authenticated validator UTxOs where possible,
- clears PostgreSQL tables used by the node projection,
- deletes ledger and mempool MPT stores,
- resets in-memory globals.

Treat `GET /reset` as destructive demo tooling, not a production API.

### Monitoring

Monitoring is enabled when the node is launched with `--with-monitoring`.

Implemented node metrics include:

- `tx_count`
- `tx_queue_size`
- `mempool_tx_count`
- `commit_block_count`
- `commit_block_tx_count`
- `commit_block_num_tx_count`
- `block_total_user_events_count`
- `total_tx_size`

The Prometheus exporter listens on `PROM_METRICS_PORT`. In the Docker Compose
runtime, Prometheus scrapes `midgard-node:9464`, Grafana runs on host port
`3001`, Loki on `3100`, Tempo on `3200` plus OTLP ports, and cAdvisor on
`8080`.

## Data Model

The PostgreSQL tables are documented in
`demo/midgard-node/DATA_STORAGE.md`. The current table set is:

- `AddressHistoryDB`
- `BlocksDB`
- `BlocksTxsDB`
- `ConfirmedLedgerDB`
- `LatestLedgerDB`
- `MempoolLedgerDB`
- `ImmutableDB`
- `MempoolDB`
- `DepositsDB`
- `TxOrdersDB`
- `WithdrawalsDB`

Most tables are instances of three reusable abstractions:

- ledger tables,
- transaction tables,
- user-event tables.

`AddressHistoryDB`, `BlocksDB`, and `BlocksTxsDB` have dedicated structures.

In addition to PostgreSQL, LevelDB-backed MPT stores persist:

- the ledger state after the newest block in `BlocksDB`,
- the mempool MPT state.

## Ledger State Views

The node keeps several ledger views:

- `MempoolLedgerDB`: most up-to-date ledger after accepted mempool
  transactions.
- `LatestLedgerDB`: ledger after the latest submitted block.
- `ConfirmedLedgerDB`: ledger of the confirmed state.
- Ledger MPT: state after the newest block commitment.

These views intentionally differ while blocks are committed but not submitted
or merged.

## Event And Block Flow

The current processing model is:

1. A client submits an L2 transaction CBOR through `POST /submit`.
2. The HTTP handler performs lightweight hex validation and queues the CBOR.
3. The transaction queue processor deserializes it and inserts it into
   `MempoolDB`.
4. L1 user-event sync periodically imports deposit, tx-order, and withdrawal
   UTxOs from Cardano.
5. Block commitment gathers eligible user events and transaction requests for
   an event interval, applies them to the ledger, computes roots, and stores an
   unsubmitted block commitment.
6. Block submission submits the L1 commitment transaction and advances latest
   ledger, immutable transaction, block-transaction, and address-history
   projections.
7. Merge processing submits state-queue merge transactions to advance confirmed
   state.

## Midgard SDK

`demo/midgard-sdk` is the off-chain transaction construction library. It
exports modules for:

- protocol parameters,
- ledger state,
- linked lists,
- initialization,
- state queue,
- scheduler,
- operator registries,
- active and retired operators,
- hub oracle,
- escape hatch,
- user events,
- settlement,
- fraud proofs.

The node imports it as:

```ts
import * as SDK from "@al-ft/midgard-sdk";
```

## Midgard TS Codec

`demo/midgard-ts` is a pure TypeScript codec package. It defines binary
encoding and decoding helpers for:

- primitive hashes, keys, addresses, and output references,
- transaction outputs,
- transactions and witness sets,
- deposit and withdrawal event info,
- block headers and block bodies.

It is useful as a protocol-shape reference, but it is not the runtime HTTP
server.

## Midgard Manager

`demo/midgard-manager` contains:

- `@midgard-manager/cli`: interactive and direct CLI commands,
- `@midgard-manager/tx-generator`: test transaction generation and submission.

The transaction generator checks node availability with `GET /tx` and submits
through `POST /submit`. The node-status command currently probes
`/api/status`, which the node does not implement.

## Related Docs

- [API](./api.md)
- [Environment](./environment.md)
