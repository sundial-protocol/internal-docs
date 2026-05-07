# Midgard Node API

This document describes the HTTP surface exposed by
`demo/midgard-node`. It is based on `demo/midgard-node/src/commands/listen.ts`
and the manager/transaction-generator clients under `demo/midgard-manager`.

## Scope

The Midgard node exposes unversioned RPC-style HTTP endpoints. It does not use
NestJS, Swagger, `/v1`, or request DTO classes.

The server is built with `@effect/platform` and started by:

```bash
cd demo/midgard-node
pnpm listen
```

or in Docker:

```bash
cd demo/midgard-node
docker compose up -d
```

By default the node listens on `PORT=3000`.

## Current Caveats

- `POST /submit` passes transaction CBOR in the query string
  (`tx_cbor=<hex>`), not in the request body.
- `POST /submit` only performs synchronous hex-shape validation before
  enqueueing. Cardano transaction deserialization happens later in the queue
  processor.
- `GET /init`, `GET /commit`, `GET /merge`, and `GET /reset` are side-effecting
  operations even though they use `GET`.
- `GET /reset` is destructive development tooling.
- `demo/midgard-manager/packages/cli/src/commands/node.ts` probes
  `/api/status`, but `midgard-node` does not currently register `/api/status`.
- There is no route today for peer discovery, chain height, formal health, or a
  versioned status response.

## Endpoint Summary

| Method | Endpoint                      | Purpose                                                                |
| ------ | ----------------------------- | ---------------------------------------------------------------------- |
| `GET`  | `/tx?tx_hash=<64-hex>`        | Look up a transaction CBOR from mempool first, then immutable storage. |
| `GET`  | `/utxos?address=<bech32>`     | Return current mempool-ledger UTxOs for an address.                    |
| `GET`  | `/txs?address=<bech32>`       | Return address-history transaction CBORs for an address.               |
| `GET`  | `/block?header_hash=<56-hex>` | Return transaction hashes mapped to a Midgard block header hash.       |
| `POST` | `/submit?tx_cbor=<hex>`       | Queue an L2 transaction CBOR for processing.                           |
| `GET`  | `/init`                       | Run initialization and genesis programs.                               |
| `GET`  | `/commit`                     | Manually trigger block commitment.                                     |
| `GET`  | `/merge`                      | Manually trigger confirmed-state merge.                                |
| `GET`  | `/reset`                      | Reset demo chain/node state and clear local projections.               |
| `GET`  | `/stateQueue`                 | Fetch and log state-queue UTxOs; return header keys.                   |
| `GET`  | `/logBlocksTxsDB`             | Log block-to-transaction counts.                                       |
| `GET`  | `/logGlobals`                 | Log in-memory global state.                                            |

## Client Flow

### 1. Submit an L2 transaction

The transaction generator calls:

```http
POST /submit?tx_cbor=<hex-encoded-cardano-transaction>
```

Success response:

```json
{
  "message": "Successfully added the transaction to the queue"
}
```

Invalid hex response:

```json
{
  "error": "Invalid CBOR provided"
}
```

After enqueueing, the background transaction queue processor:

1. drains queued CBOR strings,
2. deserializes each value as a Cardano transaction,
3. computes the transaction hash,
4. extracts spent inputs and produced outputs,
5. inserts the processed transaction into `MempoolDB`.

### 2. Check whether the node is available

The transaction generator treats this lookup as an availability check:

```http
GET /tx?tx_hash=0000000000000000000000000000000000000000000000000000000000000000
```

If the node is up and the dummy transaction is absent, the expected response is
HTTP `404`. If the request cannot connect, the generator treats the node as
unavailable and stores generated transactions locally.

### 3. Look up a transaction

```http
GET /tx?tx_hash=<64-hex-transaction-hash>
```

Validation:

- `tx_hash` must be a string,
- it must be hex,
- it must be exactly 64 hex characters.

Lookup order:

1. `MempoolDB`
2. `ImmutableDB`

Success response:

```json
{
  "tx": "<hex-cbor>"
}
```

Not found response:

```json
{
  "error": "Transaction not found: <tx_hash>"
}
```

Invalid request response:

```json
{
  "error": "Invalid transaction hash: <tx_hash>"
}
```

### 4. Query spendable L2 UTxOs for an address

```http
GET /utxos?address=<cardano-bech32-address>
```

Validation:

- `address` must be a string,
- it must parse through Lucid's `getAddressDetails`,
- it must have a payment credential.

Data source:

- `MempoolLedgerDB.retrieveByAddress(...)`

Success response:

```json
{
  "utxos": [
    {
      "outref": "<hex-cbor-output-reference>",
      "value": "<hex-cbor-transaction-output>"
    }
  ]
}
```

Invalid request response:

```json
{
  "error": "Invalid address: <address>"
}
```

### 5. Query address transaction history

```http
GET /txs?address=<cardano-bech32-address>
```

Validation is the same as `/utxos`.

Data source:

- `AddressHistoryDB.retrieve(...)`

Success response:

```json
{
  "txs": ["<hex-cbor>", "<hex-cbor>"]
}
```

### 6. Query transactions included in a block

```http
GET /block?header_hash=<56-hex-header-hash>
```

Validation:

- `header_hash` must be a string,
- it must be hex,
- it must be exactly 56 hex characters.

Data source:

- `BlocksTxsDB.retrieveTxHashesByHeaderHash(...)`

Success response:

```json
{
  "hashes": ["<64-hex-transaction-hash>"]
}
```

Invalid request response:

```json
{
  "error": "Invalid block hash: <header_hash>"
}
```

## Operator And Debug Operations

These endpoints mutate node or on-chain state. They are useful for demo and
local operation, but they are not actor-facing product APIs.

### Initialize

```http
GET /init
```

Runs:

- `Initialization.program`
- `Genesis.program`

Success response:

```json
{
  "message": "Initiation successful: <tx_hash>"
}
```

The node also runs `Genesis.program` during startup.

### Commit

```http
GET /commit
```

Runs `blockCommitmentAction`, which starts the block commitment worker once.

Success response:

```json
{
  "message": "Block commitment successful: <result>"
}
```

### Merge

```http
GET /merge
```

Runs `mergeAction`, which builds and submits a state-queue merge transaction.

Success response:

```json
{
  "message": "Merging confirmed state successful: <result>"
}
```

### Reset

```http
GET /reset
```

Runs `Reset.program`.

Effects:

- sets `RESET_IN_PROGRESS`,
- spends and burns Midgard authenticated validator UTxOs where possible,
- clears node PostgreSQL projections,
- deletes ledger and mempool MPT stores,
- resets in-memory globals.

Success response:

```json
{
  "message": "Collected all UTxOs successfully!"
}
```

Treat this route as destructive local tooling.

### State queue

```http
GET /stateQueue
```

Fetches state-queue UTxOs from Cardano L1 using SDK helpers, logs a visual
state-queue representation, and returns non-empty header keys.

Success response:

```json
{
  "headers": ["<header-hash>"]
}
```

### Log block-to-transaction table

```http
GET /logBlocksTxsDB
```

Logs grouped `BlocksTxsDB` counts and returns:

```json
{
  "message": "BlocksTxsDB drawn in server logs!"
}
```

### Log globals

```http
GET /logGlobals
```

Logs:

- `BLOCKS_IN_QUEUE`,
- `LATEST_SYNC_TIME_OF_STATE_QUEUE_LENGTH`,
- `RESET_IN_PROGRESS`.

Success response:

```json
{
  "message": "Global variables logged!"
}
```

## Error Behavior

Most unexpected failures are translated to HTTP `500` JSON responses shaped
like:

```json
{
  "error": "Something went wrong"
}
```

Some handlers provide more specific messages, for example:

- `db failure with table <table>`
- `<error-tag>: <message>`
- SDK error messages from Lucid, CML, hashing, linked-list, or state-queue
  failures.

## Monitoring Endpoint

When the node is started with:

```bash
node dist/index.js listen --with-monitoring
```

the OpenTelemetry Prometheus exporter exposes metrics from a separate HTTP
server on `PROM_METRICS_PORT`, defaulting to `9464`:

```http
GET /metrics
```

In Docker Compose this is scraped as `midgard-node:9464`.

## Related Docs

- [Architecture](./architecture.md)
- [Environment](./environment.md)
