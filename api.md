# Midgard Node API

This document describes the HTTP surface exposed by
[`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node). It is based on [`sundial-node/src/commands/listen.ts`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/src/commands/listen.ts)
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

The route set actually served depends on `NODE_ROLE` (see
[Router Variants](#router-variants) below) — a monolith node (`NODE_ROLE=all`,
the default) serves every route in this document; a node running
`NODE_ROLE=api` serves a smaller subset intended for public ingress.

## Current Caveats

- `POST /submit` takes the transaction CBOR as the **raw POST body** (a plain
  hex string, not JSON, not a query parameter). An earlier version of this
  document said the CBOR travels as a `tx_cbor` query-string parameter — that
  was never accurate for the code path that actually serves the request in
  production; see [Submit Implementation Detail](#submit-implementation-detail).
- `GET /init`, `GET /commit`, `GET /merge`, and `GET /reset` are side-effecting
  operations even though they use `GET`.
- `GET /reset` is destructive development tooling, and is **not** served by
  `NODE_ROLE=api` nodes.
- `demo/midgard-manager/packages/cli/src/commands/node.ts` probes
  `/api/status`, but `midgard-node` does not register that route. Use
  `GET /health/live` / `GET /health/ready` instead.
- [`sundial-node/src/services/always-succeeds.ts`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/src/services/always-succeeds.ts) hard-codes its script
  network to `"Preprod"` regardless of the node's own `NETWORK` config value.
  If a node is ever pointed at `Mainnet`, the derived contract
  addresses/policy IDs from this service would not match what a
  `NETWORK=Mainnet` config otherwise implies — verify this before any
  non-Preprod use of the always-succeeds contracts.
- No endpoint other than `POST /faucet/claims` performs authentication or
  rate-limiting at the application layer. See
  [security.md](./security.md#no-built-in-authentication-or-rate-limiting).

## Router Variants

`listen.ts` defines two router functions; which one is bound to `PORT` depends
on `NODE_ROLE`:

- **`router`** — served when `NODE_ROLE` is `all` (the default) or unset. All
  17 routes below.
- **`apiIngressRouter`** — served when `NODE_ROLE=api`. A restricted subset:
  `GET /health/live`, `GET /health/ready`, `GET /commit`,
  `GET /stateQueue/root-unit-diagnostics`, `GET /commitment-wallet/balance`,
  `POST /submit`, `POST /faucet/claims`. Everything else — including all read
  endpoints (`/tx`, `/txs`, `/utxos`, `/block`) and the remaining
  operator/debug endpoints — is **not served** by an `api`-role node.

This split-role deployment is documented operationally in
[`sundial-node/README.md`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/README.md) (`NODE_ROLE=api` / `tx-processor` / `sequencer`
/ `all`). The practical implication: if you're building against a public,
horizontally-scaled `api`-role deployment, don't expect `/tx`, `/txs`,
`/utxos`, or `/block` to be reachable there — those are monolith-only today.

## Endpoint Summary

| Method | Endpoint                        | Routers    | Purpose                                                                |
| ------ | -------------------------------- | ---------- | ------------------------------------------------------------------------ |
| `GET`  | `/health/live`                   | full, api  | Liveness probe — always `200`.                                          |
| `GET`  | `/health/ready`                  | full, api  | Readiness probe — checks DB, Redis, and L1 provider concurrently.       |
| `GET`  | `/tx?tx_hash=<64-hex>`           | full only  | Look up a transaction CBOR from mempool first, then immutable storage.  |
| `GET`  | `/txs?address=<bech32>&limit=&offset=` | full only | Return paginated address-history transaction CBORs for an address.     |
| `GET`  | `/utxos?address=<bech32>`        | full only  | Return current mempool-ledger UTxOs for an address.                     |
| `GET`  | `/block?header_hash=<56-hex>`    | full only  | Return transaction hashes mapped to a Midgard block header hash.        |
| `POST` | `/submit`                        | full, api  | Queue an L2 transaction CBOR (raw hex body) for processing.             |
| `POST` | `/faucet/claims`                 | full, api  | Bearer-authenticated server-to-server testnet-ADA faucet claim.         |
| `GET`  | `/init`                          | full only  | Mint the state-queue root unit and run genesis programs.                |
| `GET`  | `/commit`                        | full, api  | Manually trigger one block-commitment cycle.                            |
| `GET`  | `/merge`                         | full only  | Manually trigger merging the oldest confirmed block into confirmed state. |
| `GET`  | `/reset`                         | full only  | Reset demo chain/node state and clear local projections. Destructive.   |
| `GET`  | `/stateQueue`                    | full only  | Fetch and log the state-queue linked list; return header keys.          |
| `GET`  | `/stateQueue/root-unit-diagnostics` | full, api | Report health of the state-queue root unit.                            |
| `GET`  | `/stateQueue/repair-root-units`  | full only  | Targeted repair burning duplicate state-queue root units.               |
| `GET`  | `/commitment-wallet/balance`     | full, api  | Return the block-commitment operator wallet's lovelace balance.         |
| `GET`  | `/logBlocksTxsDB`                | full only  | Log block-to-transaction counts to server logs.                        |
| `GET`  | `/logGlobals`                    | full only  | Log in-memory global state to server logs.                             |

That's 17 routes on the full router, 7 on `apiIngressRouter`, plus a separate
`GET /metrics` server (see [Monitoring Endpoint](#monitoring-endpoint)).

## Client Flow

### 1. Check whether the node is available

Prefer the health endpoints over any custom probe:

```http
GET /health/live
```

Always returns `200 {"status":"ok"}` if the process is up — use this as a
basic liveness check.

```http
GET /health/ready
```

Runs three checks concurrently, each individually bounded to 3 seconds so one
slow dependency can't hang the whole check: a Postgres `SELECT 1`, a Redis
`PING`, and the configured L1 provider's `getProtocolParameters()`. On success:
`200 {"status":"ready"}`. If any subsystem fails or times out:
`503 {"status":"not_ready","failing":["database"|"redis"|"l1Provider", ...]}`.

(Older tooling in `demo/midgard-manager`'s transaction generator instead polls
`GET /tx?tx_hash=<64 zeros>` and treats a `404` as "node is up" and a
connection error as "node is down" — that pattern still works but predates
these health endpoints and is more indirect than using them directly.)

### 2. Submit an L2 transaction

```http
POST /submit
Content-Type: text/plain

<hex-encoded-cardano-transaction-cbor>
```

The CBOR is the **entire raw request body** — not JSON, not a query
parameter. Success response:

```json
{
  "message": "Successfully added the transaction to the queue",
  "id": "<redis-stream-entry-id>"
}
```

Invalid hex response (`400`):

```json
{ "error": "Invalid CBOR provided" }
```

Enqueue failure response (`500`):

```json
{ "error": "Failed to enqueue transaction" }
```

After enqueueing, on a node running the `tx-processor` role, a background
worker pool:

1. claims/reads batches off the Redis stream (consumer group, with automatic
   reclaim of stale-pending entries),
2. parses each CBOR string into a Cardano transaction in a worker-thread pool,
3. computes the transaction hash and extracts spent inputs / produced outputs,
4. validates it (including a min-fee check) and inserts it into `MempoolDB`,
   acknowledging the stream entry on success.

A single malformed transaction is rejected and retried/dead-lettered per
`TX_QUEUE_MAX_DELIVERY_ATTEMPTS` — it does not affect other in-flight
submissions or destabilize the worker pool.

#### Submit implementation detail

Two implementations of `POST /submit` exist in the codebase, but only one
handles real traffic:

- **Live path**: `services/raw-submit-interceptor.ts` attaches directly to the
  raw Node `http.Server`, *before* `@effect/platform`'s own request listener
  is registered, and intercepts any request where `req.method === "POST"` and
  `req.url === "/submit"` exactly (no query string). It reads the body itself
  and calls Redis `XADD` directly. This is the code path documented above.
- **Effect-registered path**: `postSubmitHandler` in `listen.ts` is still
  wired into both routers via `HttpRouter.post`, but because the raw
  interceptor claims every exact `/submit` POST first, this handler is
  unreachable by real HTTP traffic. It's kept for unit-testability (exported
  as `postSubmitHandlerForTesting`) and produces the same response shapes
  described above. If you're changing submit behavior in this codebase,
  the raw interceptor is the one to change — updating only the Effect handler
  has no runtime effect.

One consequence of the `req.url === "/submit"` exact-match: a request sent to
`/submit?tx_cbor=...` (the old, incorrect documented shape) does **not** match
the interceptor and falls through to the Effect router instead, which reads
its body — not the query string — the same as the live path. Either way, put
the CBOR in the body of a plain `POST /submit`.

### 3. Look up a transaction

```http
GET /tx?tx_hash=<64-hex-transaction-hash>
```

Validation: `tx_hash` must be a string, hex, and exactly 64 characters.

Lookup order: `MempoolDB`, then `ImmutableDB`.

Success response:

```json
{ "tx": "<hex-cbor>" }
```

Not found (`404`):

```json
{ "error": "Transaction not found: <tx_hash>" }
```

Invalid request (`400`):

```json
{ "error": "Invalid transaction hash: <tx_hash>" }
```

### 4. Query spendable L2 UTxOs for an address

```http
GET /utxos?address=<cardano-bech32-address>
```

Validation:

- `address` must be a string, or the response is
  `400 {"error":"Invalid address type: <address>"}`.
- It must parse through Lucid's `getAddressDetails` and have a payment
  credential, or the response is
  `400 {"error":"Invalid address format: <address>"}`.
- Any other parse failure returns
  `400 {"error":"Invalid address: <address>"}`.

Data source: `MempoolLedgerDB.retrieveByAddress(...)`.

Success response:

```json
{
  "utxos": [
    { "outref": "<hex-cbor-output-reference>", "value": "<hex-cbor-transaction-output>" }
  ]
}
```

### 5. Query address transaction history

```http
GET /txs?address=<cardano-bech32-address>&limit=<n>&offset=<n>
```

Address validation is the same three-tier scheme as `/utxos` above.

- `limit` is optional. It must be a non-negative integer ≥ 1 if present.
  Default is `100`; values above `500` are silently clamped to `500`
  (`AddressHistoryDB.DEFAULT_ADDRESS_HISTORY_LIMIT` /
  `MAX_ADDRESS_HISTORY_LIMIT`). An invalid `limit` returns
  `400 {"error":"Invalid limit: <value>"}`.
- `offset` is optional, must be a non-negative integer if present, defaults to
  `0`. An invalid `offset` returns `400 {"error":"Invalid offset: <value>"}`.

Data source: `AddressHistoryDB.retrieve(address, { limit, offset })`.

Success response:

```json
{
  "txs": ["<hex-cbor>", "<hex-cbor>"],
  "limit": 100,
  "offset": 0,
  "hasMore": true
}
```

`hasMore` is `true` when exactly `limit` rows were returned (i.e. there may be
more beyond this page) — page forward by re-requesting with
`offset + limit`.

### 6. Query transactions included in a block

```http
GET /block?header_hash=<56-hex-header-hash>
```

Validation: `header_hash` must be a string, hex, and exactly 56 characters.

Data source: `BlocksTxsDB.retrieveTxHashesByHeaderHash(...)`.

Success response:

```json
{ "hashes": ["<64-hex-transaction-hash>"] }
```

Invalid request (`400`):

```json
{ "error": "Invalid block hash: <header_hash>" }
```

### 7. Claim testnet ADA from the faucet

```http
POST /faucet/claims
Authorization: Bearer <FAUCET_API_KEY>
Content-Type: application/json

{ "address": "<bech32>", "idempotencyKey": "<string>", "ipHash": "<string>" }
```

This is a server-to-server endpoint, not a public one — see
[security.md](./security.md) for the auth/rate-limit caveats before exposing
it publicly.

- If `FAUCET_ENABLED` is false or `FAUCET_API_KEY` is empty, the route returns
  `404 {"error":"Faucet is not enabled"}` — deliberately indistinguishable
  from a non-existent route.
- A missing or incorrect bearer token returns `401 {"error":"Unauthorized"}`.
- `address`, `idempotencyKey`, and `ipHash` must all be non-empty strings, or
  `400`.

Success response:

```json
{
  "claimId": "<uuid>",
  "txHash": "<hex>",
  "amount": "<lovelace-as-string>",
  "nextEligibleAt": "<ISO-8601-timestamp>"
}
```

Error responses carry a stable `code` alongside `error` and an HTTP status:

| `code`                          | Status | Meaning                                             |
| -------------------------------- | ------ | ---------------------------------------------------- |
| `DISABLED`                       | 404    | Faucet not enabled.                                  |
| `ADDRESS_INVALID`                | 400    | Address failed to parse.                             |
| `ADDRESS_NETWORK_MISMATCH`       | 400    | Address is for the wrong network.                    |
| `ADDRESS_NO_PAYMENT_CREDENTIAL`  | 400    | Address has no payment credential.                   |
| `ADDRESS_SCRIPT`                 | 400    | Address is a script address, not a wallet address.   |
| `COOLDOWN`                       | 429    | This address claimed too recently. Body includes `nextEligibleAt`. |
| `IP_LIMIT`                       | 429    | Daily per-IP claim cap reached.                      |
| `DEPLETED`                       | 503    | Faucet wallet has insufficient funds.                |
| `VALIDATION_FAILED`              | 500    | Transaction failed to validate.                      |
| `INTERNAL`                       | 500    | Unexpected internal error.                            |

## Operator And Debug Operations

These endpoints mutate node or on-chain state. They are useful for demo and
local operation, but they are not actor-facing product APIs, and most of them
are excluded from `apiIngressRouter` (see [Router Variants](#router-variants)).

### Initialize

```http
GET /init
```

Guarded by the same in-memory lock as `/reset` and `/stateQueue/repair-root-units`
(`Globals.RESET_IN_PROGRESS`) — a concurrent request returns
`409 {"error":"Reset already in progress"}`.

If the state-queue root unit already exists, the node refuses to mint a
second one:

```json
{
  "error": "State queue already initialized (or dirty). Refusing to mint another root unit.",
  "rootUnit": "<policy-id+asset-name>",
  "stateQueueAddress": "<bech32>",
  "count": 1,
  "outRefs": ["<txHash>#<outputIndex>"]
}
```
(`409`)

Otherwise, runs `Initialization.program` (mints the root unit) then
`Genesis.program` (seeds local L2 ledger rows and submits a first
deposit + tx-order pair on non-Mainnet networks). Success response:

```json
{ "message": "Initiation successful: <tx_hash>" }
```

The node also runs `Genesis.program` once at startup on any node running the
sequencer role.

### Commit

```http
GET /commit
```

Runs `blockCommitmentAction`, which starts the block commitment worker once.
Available on both routers.

```json
{ "message": "Block commitment successful: <result>" }
```

### Merge

```http
GET /merge
```

Runs `mergeAction`, which builds and submits a state-queue merge transaction.

```json
{ "message": "Merging confirmed state successful: <result>" }
```

### Reset

```http
GET /reset
```

Lock-guarded the same way as `/init`; concurrent calls get
`409 {"error":"Reset already in progress"}`. Runs `Reset.program`:

- spends and burns Midgard authenticated validator UTxOs where possible,
- clears node PostgreSQL projections,
- deletes ledger and mempool MPT stores on disk,
- resets in-memory globals.

```json
{ "message": "Collected all UTxOs successfully!" }
```

Treat this route as destructive local tooling. It is not served by
`NODE_ROLE=api` nodes.

### State queue

```http
GET /stateQueue
```

Fetches state-queue UTxOs from Cardano L1 via the SDK, logs a visual
state-queue representation, and returns non-empty header keys.

```json
{ "headers": ["<header-hash>"] }
```

### State-queue root-unit diagnostics

```http
GET /stateQueue/root-unit-diagnostics
```

Available on both routers. Reports whether the state queue has exactly one
root unit (the healthy state):

```json
{
  "status": "ok",
  "resetInProgress": false,
  "stateQueueAddress": "<bech32>",
  "rootUnit": "<policy-id+asset-name>",
  "count": 1,
  "outRefs": ["<txHash>#<outputIndex>"]
}
```

`status` is `"invalid"` when `count !== 1` (zero or duplicate root units). On
a provider/query failure, returns `503` with `status: "error"` and a `cause`
string instead of throwing.

### State-queue root-unit repair

```http
GET /stateQueue/repair-root-units
```

Lock-guarded like `/init`/`/reset`. Runs a targeted repair that burns
duplicate state-queue root units when `count > 1` — faster recovery than a
full `/reset` for deep historical state. Successful repair brings the root
unit count to `0`; run `GET /init` once afterward to mint a fresh single root
unit.

```json
{ "message": "State-queue root-unit repair completed successfully!" }
```

### Commitment wallet balance

```http
GET /commitment-wallet/balance
```

Available on both routers. Derives the block-commitment operator wallet
address from `L1_OPERATOR_SEED_PHRASE_FOR_BLOCK_COMMITMENT` and queries its
UTxOs (bounded to a 4-second timeout).

```json
{ "lovelaceBalance": "<bigint-as-string>" }
```

Useful for alerting when the commitment wallet is running low.

### Log block-to-transaction table

```http
GET /logBlocksTxsDB
```

Logs grouped `BlocksTxsDB` counts to server logs (not the response body):

```json
{ "message": "BlocksTxsDB drawn in server logs!" }
```

### Log globals

```http
GET /logGlobals
```

Logs `BLOCKS_IN_QUEUE`, `LATEST_SYNC_TIME_OF_STATE_QUEUE_LENGTH`, and
`RESET_IN_PROGRESS` to server logs:

```json
{ "message": "Global variables logged!" }
```

## Error Behavior

Most unexpected failures are translated to HTTP `500` JSON responses shaped
like:

```json
{ "error": "Something went wrong" }
```

Some handlers provide more specific messages, for example:

- `db failure with table <table>`
- `<error-tag>: <message>`
- SDK error messages from Lucid, CML, hashing, linked-list, or state-queue
  failures.

Router-level catches (`Effect.catchAllCause`) mean an entirely unanticipated
failure anywhere in a handler still comes back as a `500` JSON body rather
than an unhandled connection drop.

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

This is a plain `PrometheusExporter` HTTP server, not part of either
`@effect/platform` router above — it exists independently of `NODE_ROLE`. In
Docker Compose this is scraped as `midgard-node:9464` (or role-specific ports
in split deployments — see [`sundial-node/README.md`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/README.md)).

## Related Docs

- [Architecture](./architecture.md)
- [Environment](./environment.md)
- [Smart Contract Interactions](./smart-contracts.md)
- [Security Best Practices](./security.md)
