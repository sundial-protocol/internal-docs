# Smart Contract Interactions

This document explains how [`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node) and [`sundial-sdk`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-sdk) relate
to the actual Midgard on-chain contracts — which validators exist, which ones
this demo deployment actually runs against, and how a transaction moves from
an HTTP submission to an L1 commitment.

Read this before assuming that anything accepted by [`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node) has
been validated on-chain. It has not, in this environment. See
[The Demo Runs Against Placeholder Validators](#the-demo-runs-against-placeholder-validators)
below.

## Three Representations Of The Protocol

The repository contains three distinct representations of Midgard, at three
different levels of "real":

| Layer | Location | What it is |
| --- | --- | --- |
| Formal spec | `technical-spec/Lean4Midgard/` (submodule, `github.com/input-output-hk/rnd-midgard`) | IOG's Lean4 formal verification of the protocol's state machines (StateQueue, Scheduler, OperatorDirectory, Settlement, Bridge/user-events, fraud-proof catalogue). A research artifact and correctness reference, not executable Cardano code. |
| Real implementation | `onchain/aiken/` (validators + library code), `onchain/plutarch/` (parallel Haskell/Plutarch implementation) | The actual Aiken/Plutus V3 contracts with real validation logic: linked-list state queue, operator directory, scheduler, multi-step fraud proofs, settlement, user-events. |
| What this demo runs | [`sundial-node/blueprints/always-succeeds`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node/blueprints/always-succeeds) | A compiled Aiken project where every validator unconditionally returns `True`. |

[`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node) only ever talks to the third layer. It never references
`onchain/aiken` at all.

## The Demo Runs Against Placeholder Validators

[`sundial-node/src/services/always-succeeds.ts`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-node/src/services/always-succeeds.ts) loads
`blueprints/always-succeeds/plutus.json` (the compiled output of
`blueprints/always-succeeds/validators/{midgard,fraud-proofs}.ak`) and builds
every validator address/policy ID the node uses from it. Every validator in
that file has the same shape:

```aiken
validator state_queue_spend {
  else(_) {
    trace @"Midgard Demo – State Queue Spend"
    True
  }
}
```

This is true of all 17 validator pairs (`hub_oracle`, `scheduler`,
`state_queue`, `registered_operators`, `active_operators`,
`retired_operators`, `escape_hatch`, `fraud_proof_catalogue`, `fraud_proof`,
`deposit`, `reserve`, `payout`, `withdrawal`, `tx_order`, `settlement`, plus
the 4 fraud-proof validators) — `blueprints/always-succeeds/README.md` is, in
fact, Aiken's unmodified `aiken new` scaffold README, which is a good tell
that nobody was meant to mistake this for production logic.

**What this means concretely:** in this demo deployment, Cardano script
validation enforces nothing about Midgard's protocol rules — not fee
correctness, not UTxO validity, not double-spend prevention, not fraud-proof
soundness. Every one of those properties is enforced entirely by
[`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node)'s and [`sundial-sdk`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-sdk)'s own off-chain code (mempool
validation, the block-commitment worker, etc.). A transaction or datum
shaped correctly enough to satisfy the SDK's TypeScript types will also
satisfy the on-chain script, regardless of whether it's actually valid under
the protocol's real rules — because the script never looks.

Practical implications:

- **Never point real value at this demo's contract addresses.** There is no
  on-chain backstop. This applies to any environment running the
  always-succeeds blueprint, testnet included.
- Treat everything [`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node) enforces as *application-level*
  correctness, not protocol-level correctness — see
  [security.md](./security.md) for what that means for integrators.
- Don't use behavior observed against this demo as evidence that the real
  Midgard protocol (`onchain/aiken`) would accept or reject the same
  transaction — the real validators are not in the loop here at all.

One further caveat: `always-succeeds.ts` hard-codes
`const NETWORK: Network = "Preprod"` for deriving script addresses,
independent of the node's own `NodeConfig.NETWORK`. Confirm this before
relying on always-succeeds-derived addresses outside Preprod.

## SDK-To-Validator Map

[`sundial-sdk`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-sdk) (`@al-ft/midgard-sdk`) is the off-chain TypeScript library
for building transactions that target these validators — the "Midgard
Typescript library for building operator and watcher transactions" per its
own README. It has no HTTP server, no database, no queues; it's a pure
tx-building/query library that takes a `LucidEvolution` instance and returns
`Effect` blueprints (`unsignedXTxProgram` / `incompleteXTxProgram`, per the
Lucid Evolution naming convention).

The table below maps each public SDK module (as exported from
[`sundial-sdk/src/index.ts`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-sdk/src/index.ts)) to the demo's always-succeeds stub and, where
one exists, the real `onchain/aiken` implementation:

| SDK module | Demo stub validator(s) | Real `onchain/aiken` implementation |
| --- | --- | --- |
| `state-queue.ts` | `state_queue_mint` / `state_queue_spend` | `validators/state-queue.ak`, `lib/midgard/state-queue.ak` |
| `initialization.ts` | `state_queue_mint` (genesis path) | `state-queue.ak`'s genesis/`Init` redeemer |
| `scheduler.ts` | `scheduler_mint` / `scheduler_spend` | `validators/scheduler.ak`, `lib/midgard/scheduler.ak` |
| `hub-oracle.ts` | `hub_oracle_mint` / `hub_oracle_spend` | `lib/midgard/hub-oracle.ak` — a shared library read as a **reference input** by other real validators, not a standalone top-level validator in `validators/` |
| `active-operators.ts` | `active_operators_mint` / `active_operators_spend` | `validators/operator-directory/active-operators.ak` |
| `registered-operators.ts` | `registered_operators_mint` / `registered_operators_spend` | `validators/operator-directory/registered-operators.ak` |
| `retired-operators.ts` | `retired_operators_mint` / `retired_operators_spend` | `validators/operator-directory/retired-operators.ak` |
| `user-events/deposit.ts` | `deposit_mint` / `deposit_spend` | `validators/user-events/deposit.ak` |
| `user-events/withdrawal.ts` | `withdrawal_mint` / `withdrawal_spend` | `validators/user-events/withdrawal.ak` (also uses `lib/midgard/payout.ak` for payout logic) |
| `user-events/tx-order.ts` | `tx_order_mint` / `tx_order_spend` | `validators/user-events/tx-order.ak` |
| `fraud-proof/*.ts` | `fraud_proof_mint` / `fraud_proof_spend`, `fraud_proof_catalogue_mint` / `fraud_proof_catalogue_spend` | `validators/fraud-proof.ak`, `validators/fraud-proof-catalogue.ak`, plus the multi-step programs under `validators/fraud-proofs/{double-spend,invalid-range,non-existent-input,non-existent-input-no-index}/step-0N.ak` |
| `escape-hatch.ts` | `escape_hatch_mint` / `escape_hatch_spend` | Not found under `onchain/aiken` at the time of writing — verify before assuming real logic exists here. |
| — (no SDK module; `reserve` is only referenced in `always-succeeds.ts` and `genesis.ts`) | `reserve_spend` / `reserve_withdraw` | Not found. `genesis.ts` comments: *"Reserve is not entirely defined in specs. This might change."* |
| — (not exported from SDK `index.ts`, though the source file exists) | `settlement_mint` / `settlement_spend` | `validators/settlement.ak`, `lib/midgard/settlement.ak`. [`sundial-sdk/src/settlement.ts`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-sdk/src/settlement.ts) exists but isn't re-exported from `index.ts` — the package has no `exports` map in `package.json` (only `main`/`types`), so this module isn't part of the supported public surface today. |

Two things worth calling out beyond the table:

- **`computation-thread.ak`** (`validators/computation-thread.ak`,
  `lib/midgard/computation-thread.ak`) is real on-chain logic with no
  counterpart anywhere in the demo's `AlwaysSucceedsContract` /
  `MidgardValidators` type. The demo doesn't model computation threads at
  all today.
- **`internals.ts`** (in both [`sundial-sdk/src`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-sdk/src) and
  [`sundial-sdk/src/user-events`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-sdk/src/user-events)) is genuinely internal — shared
  coercion/validation helpers used by the modules above, not a protocol
  operation of its own.

## Transaction Lifecycle

This is the path a transaction takes from `POST /submit` to being folded into
confirmed L1 state. See [api.md](./api.md#2-submit-an-l2-transaction) for the
HTTP contract itself.

1. **Ingress** — `POST /submit`'s raw hex CBOR is pushed onto a Redis Stream
   (`services/redis-streams-tx-ingress-queue.ts`), with a consumer group for
   durable, at-least-once processing and a dead-letter stream for entries
   that exceed `TX_QUEUE_MAX_DELIVERY_ATTEMPTS`.
2. **Parse & validate** — on a node running the `tx-processor` role,
   `fibers/tx-queue-processor.ts` claims batches off the stream, parses each
   CBOR string into a Cardano transaction in a worker-thread pool
   (`fibers/tx-parse-worker-pool.ts` → `workers/tx-parse.ts`), and validates
   it (including a minimum-fee check) before inserting into `MempoolDB`
   (`database/mempool.ts`).
3. **L1 user events** — concurrently, on a node running the `sequencer` role,
   `fibers/sync-user-events.ts` polls L1 (via the configured provider) for
   new deposit/withdrawal/tx-order UTxOs at the (always-succeeds) event
   validator addresses and records them locally.
4. **Block commitment** — `fibers/block-commitment.ts` periodically spawns a
   worker (`workers/block-commitment.ts`) that assembles a new L2 block from
   pending mempool transactions plus synced user events within a commitment
   window, updates the ledger Merkle-Patricia-Trie
   (`workers/utils/mpt.ts`), and produces a commitment.
5. **Block submission** — `fibers/block-submission.ts` signs and submits the
   commitment transaction against the `state_queue` validator using the
   block-commitment operator wallet, with an L1 idempotency pre-check to
   avoid double-submitting after a timeout.
6. **Merge** — `fibers/merge.ts` periodically folds the oldest committed
   block into `ConfirmedState` via the state queue's `MergeToConfirmedState`
   redeemer (`transactions/state-queue/merge-to-confirmed-state.ts`), using
   the merge operator wallet.

`GET /commit` and `GET /merge` let you trigger steps 4 and 6 manually instead
of waiting for their scheduled interval — see
[api.md](./api.md#operator-and-debug-operations).

## Genesis And Bootstrap

Standing up a fresh instance involves two separate programs, both invoked
together from `GET /init`:

- **`transactions/initialization.ts`** mints the state-queue root unit — the
  beacon UTxO that anchors the linked-list state queue — building the
  fraud-proof-catalogue Merkle-Patricia-Forestry tree in the process. This is
  the closest thing to "deploying the protocol," though note that the
  Plutus scripts themselves don't need a deployment step; only their
  first authenticated UTxO does.
- **`genesis.ts`**'s `program` does two more things, both skipped on
  `NETWORK=Mainnet`:
  - seeds fake L2 ledger rows directly into `MempoolLedgerDB` from
    `NodeConfig.GENESIS_UTXOS` (funded to addresses derived from the
    `TESTNET_GENESIS_WALLET_SEED_PHRASE_{A,B,C}` env vars), bypassing any
    real deposit — a devnet/testnet convenience, not a real user flow;
  - submits one real L1 transaction: a composed genesis deposit (hard-coded
    10 ADA) + tx-order pair, so the rest of the pipeline (user-event sync →
    commitment) has something to process on a freshly initialized node.

`genesis.ts`'s `program` also runs once automatically at startup on any node
running the sequencer role, independent of `/init`.

## Using The SDK Directly

If you're building your own operator or watcher tooling rather than going
through [`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node)'s HTTP API, [`sundial-sdk`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-sdk) is the intended
entry point — install it as described in
[`sundial-sdk/README.md`](https://github.com/sundial-protocol/sundial-monorepo/blob/main/demo/midgard-sdk/README.md) (it's distributed
as a `pnpm repack` tarball today, not a published npm package) and construct
your own `LucidEvolution` instance to pass into its `Program` functions.

This is the same library [`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node) itself uses internally
(`import * as SDK from "@al-ft/midgard-sdk"` throughout `src/`), so anything
the node's HTTP API can trigger, the SDK can build directly — with the same
[always-succeeds caveat](#the-demo-runs-against-placeholder-validators)
applying regardless of which path you use.

## Related Docs

- [API](./api.md)
- [Architecture](./architecture.md)
- [Environment](./environment.md)
- [Security Best Practices](./security.md)
