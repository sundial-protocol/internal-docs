# Data Handling Addendum

Generated: 2026-08-17

- **Current node storage** - This document describes the data persisted by
  `demo/midgard-node` today: PostgreSQL protocol tables, Redis ingress streams,
  LevelDB-backed MPT state, and operational logs/metrics/traces emitted by the
  node stack.
- **Testnet vs mainnet** - Switching `NETWORK` from a testnet value to
  `Mainnet` changes network validation and disables testnet-only genesis/faucet
  seeding, but it does not add new Midgard-node data flows. In particular,
  `demo/midgard-node` does not store KYC, AML, customer profile, or onboarding
  records.
- **Boundary** - Any KYC/onboarding layer, if introduced outside this package,
  is outside the storage model documented here and must have its own data
  handling review.
- **Personal-data surface** - The node stores blockchain-derived addresses,
  transaction/UTxO CBOR, event metadata, and faucet claim records when the
  faucet is enabled. Faucet claim records store a recipient address,
  idempotency key, caller-provided IP hash, claim amount, transaction id, and
  timestamps; the node does not persist raw IP addresses in this table.

## Data Handling Determination

Sundial has a defined current data-handling boundary for `demo/midgard-node`.

The current node data scope is limited to protocol and operational data:
blockchain-derived addresses, transaction and event data, queue payloads,
ledger-state projections, and operational telemetry. The current node scope
does not include KYC, AML, customer profile, or onboarding records.

## Database Tables

We currently have the following tables:

- AddressHistoryDB
- BlocksDB
- BlocksTxsDB
- ConfirmedLedgerDB
- LatestLedgerDB
- MempoolLedgerDB
- ImmutableDB
- MempoolDB
- DepositsDB
- FaucetClaimsDB
- TxOrdersDB
- WithdrawalsDB

Apart from `AddressHistoryDB`, `BlocksDB`, `BlocksTxsDB`, and
`FaucetClaimsDB`, the rest are instances of three abstract tables:

- Ledger
- Transactions
- User events

Additionally, a LevelDB on disk stores the Midgard ledger after the latest block
commitment (detailed further [below](#ledger)). Redis Streams store submitted
transaction CBOR while requests are waiting to be processed, plus dead-letter
entries for messages that exceed the retry limit.

### Address History

For storing events and their corresponding addresses involved. Without this
table, querying interaction history of a given address would be very expensive.

This table maps "event IDs" to addresses, with 2 flags: one for the state of a
given record, and another for specifying the nature of its event.

Addresses can send/receive funds via 3 different interactions:

1. Transactions (either requests or orders)
2. Deposits
3. Withdrawals

Each entry can also have 3 different states:

1. **Slated**: For events that are not included in a block commitment. This is
   currently only used for transactions in `MempoolDB`.
2. **Submitted**: For events that are included in a submitted block commitment.
3. **Merged**: For events included in a block that's been merged into the
   confirmed state.

To add an entry based on a given transaction, first the inputs and outputs of
the transaction must be extracted. Next, the spent inputs must be resolved using
the most up-to-date ledger (which is `MempoolLedgerDB`) so that involved
addresses can be determined. The "event ID" will be the hash of the given
transaction.

For withdrawals, similar to transactions, first their spent input must be
resolved to extract the involved address. For "event ID," the withdrawal ID is
used.

Deposits are simpler as they don't need resolving. Their involved addresses are
extracted from their "event info" field.

It is of note that we are currently treating transaction requests and orders
identically. That is, while the event ID of a transaction order is different
than the hash of the transaction they carry, in this table we are ignoring their
event IDs and simply recording their transaction hashes as their identifiers.

### Blocks

Stores various information for individual block commitments. It primarily maps
header hashes to signed L1 transactions (appending to the state queue). Since
we are not submitting the transactions, the wallet state, along with the
produced UTxOs in each transaction need to be explicitly recorded in each row,
so that the consequent block commitments can be built without waiting for
on-chain confirmation.

Similar to `AddressHistoryDB`, each entry here is also marked with a status.
Namely:

- **Unsubmitted**: Block commitments in queue for submission.
- **Submitting**: Block commitments marked as in-flight before L1 interaction,
  so restart recovery can distinguish never-tried from already-tried
  submissions.
- **Submitted**: Block commitments that are successfully submitted on L1.
- **Confirmed**: Block commitments that have become available on-chain. This is
  needed for knowing when to submit the next unsubmitted commitment transaction.
- **Merged**: Block commitments that are merged into the confirmed state.

Each entry insertion is accompanied by updating relevant `AddressHistoryDB`
entries' statuses (or inserting new entries for user events).

### Blocks Transactions

In order to be able to query transactions included in a block, this table maps
header hashes to transaction hashes. This happens along with block commitments
(i.e. entry insertion into `BlocksDB`).

### Ledger

We have four ledgers, 3 of which are instances of the `Ledger` abstraction:

- **`MempoolLedgerDB`**: The most up-to-date ledger, which is populated along
  with `MempoolDB`.
- **`LatestLedgerDB`**: Ledger after the latest submitted block.
- **`ConfirmedLedgerDB`**: Ledger of the confirmed state (root element of the
  state queue).

The fourth one, the Merkle Patricia Trie (MPT), is a LevelDB instance, which
represents the ledger state after the newest entry of `BlocksDB`. This means, if
unsubmitted blocks do exist, this ledger differs from `LatestLedgerDB`. They are
identical if all blocks are submitted.

### Transactions

Two transaction tables exist:

- **`ImmutableDB`**: For storing a historical record of transactions.
- **`MempoolDB`**: Temporary storage of transactions slated for inclusion in the
  next block commitment.

Transaction hashes (i.e. IDs), are mapped to their preimages (CBOR bytes).
`MempoolDB` also stores normalized transaction effects: transaction size, spent
outrefs, produced outrefs, produced outputs, and produced addresses.

### User Events

There are 3 user event tables:

- **`DepositsDB`**
- **`TxOrdersDB`**
- **`WithdrawalsDB`**

They all record a few values:

- Event IDs: Nonce spent output references at the time of creating the events.
- Event infos: CBOR bytes of the corresponding data stored in datum of their L1
  UTxOs.
- L1 UTxOs' asset names: Blake2b256 hashes of the event IDs.
- CBOR encoded L1 UTxOs.
- Events' inclusion times.

Insertions occur at the time of discovery. However, `AddressHistoryDB` won't be
updated as it'd complicate handling cases where currently non-existent UTxOs are
spent (TODO?).

### Faucet Claims

`FaucetClaimsDB` records successful faucet claims when `FAUCET_ENABLED=true`.
It stores:

- Claim IDs.
- Idempotency keys.
- Recipient Cardano addresses.
- Caller-provided IP hashes for rate limiting.
- Claim amounts in lovelace.
- Faucet transaction IDs.
- Claim creation times.
- Next eligible claim times for address cooldown enforcement.

`NETWORK=Mainnet` disables testnet-only genesis UTxO seeding, including faucet
genesis funding. The faucet table may still exist because database
initialization creates the schema, but mainnet mode does not add a new faucet or
onboarding data flow.

## Non-PostgreSQL Storage

### Redis Ingress Streams

`POST /submit` enqueues transaction CBOR into the configured Redis stream under
the `tx_cbor` field. The tx processor consumes the stream with a Redis consumer
group. Messages that reach `TX_QUEUE_MAX_DELIVERY_ATTEMPTS` are copied to the
configured dead-letter stream with:

- Original stream name.
- Original Redis message ID.
- Transaction CBOR.
- Delivery count.
- Failure timestamp in milliseconds.
- Failure reason.

### MPT LevelDB

The node uses LevelDB-backed Merkle Patricia Tries at `LEDGER_MPT_DB_PATH` and
`MEMPOOL_MPT_DB_PATH`. These stores contain MPT keys and values derived from
transaction inputs/outrefs and transaction outputs. In the AWS deployment, this
state is mounted on encrypted EFS.

### Operational Telemetry

The node emits logs, Prometheus metrics, and OpenTelemetry traces for health,
queue depth, validation, commitment, submission, and merge behavior. Local
Compose includes Prometheus, Loki, Grafana, Tempo, and cAdvisor for development
observability; the AWS deployment configures separate observability services.
