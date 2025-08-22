# Sundial Watcher Toolkit

The Sundial Watcher Toolkit is a comprehensive set of tools and resources designed to empower users to set up and manage their own watcher services within the Sundial protocol. These tools should follow a set of standards that ensure interoperability between different watcher implementations, allowing for a diverse and resilient watcher ecosystem.

There are 4 types of watcher roles available:

- Provers
- Archivists
- Facilitators
- Canaries

## Provers

Provers watch the state queue & data availability layer (DAL) for invalid transactions or blocks. If they detect any issues, they can submit a proof to the appropriate on-chain contract and claim the block producer's deposit as a reward.

For each type of Fraud Proof (listed below), the prover service should do the following as a self contained service without user intervention:

- Monitor the state queue & DAL for the specific condition
- Generate the necessary proof data
- Submit the proof to the correct on-chain contract
- Submit subsequent transactions to continue the proof process
- Terminate the proof process if interrupted (i.e. some error occurs)
- Claim the reward if the proof is successful

User intervention is only required to start the prover service and to configure the necessary credentials and connections for submitting transactions on-chain.

A modular example implementation will be provided as part of this toolkit, ready to either be deployed as-is, or used as a reference for building a custom implementation.

### Fraud Proof Types

- Inputs
  - **NO-INPUT**: A transaction 𝑡 attempted to spend the UTxO 𝑖 that does not exist or was spent in a previous block.
  - **INPUT-NO-IDX**: A transaction 𝑡 attempted to spend the input 𝑖, which did not exist at all.
  - **WITHDRAWN-INPUT**: A transaction 𝑡 attempted to spend the input 𝑖, which was spent in a withdraw transaction.
  - **DOUBLE-SPEND**: A transaction 𝑡 attempted to spend the input 𝑖, which was spent in another transaction.
  - **DOUBLE-WITHDRAW**: A withdrawal 𝑊𝑛 attempted to spend the same input, which was already withdrawn by 𝑊1.
  - **ZERO-INPUT**: A transaction 𝑡 is in the ledger, while |inputs(𝑡)| = 0
- Validity Range
  - **INVALID-RANGE**: A transaction 𝑡 has a time-validity range that does not overlap with its block’s event interval.
- Fees
  - **MIN-FEE**: A transaction 𝑡 is in the ledger, while fee(𝑡) < min fee(𝑡).
- Signatures
  - **MISSING-REQ-SIGNER**: A transaction 𝑡 does not show a required signer.
  - **NON-REQ-SIGNER**: A transaction 𝑡 shows an additional, non-required signer.
  - **INVALID-SIGNER**: A transaction 𝑡 has an invalid signer.
  - **MISSING-SIGNATURE**: A required signature, corresponding to 𝑠 is missing.
- Native Scripts
  - **MISSING-NATIVE-SCRIPT**: A required script, corresponding to 𝑠 is missing.
  - **NATIVE-SCRIPT-INVALID**: A native script 𝑠 validation fails.
- Value
  - **VALUE-NOT-PRESERVED**: A transaction 𝑡 does not preserve value.
  - **ADA-MINTED**: A transaction 𝑡 mints ADA.
  - **NEGATIVE-OUTPUT-VALUE**: A transaction 𝑡 has an output with a negative value.
  - **MIN-SAT-TX**: An output of a transaction 𝑡 does not satisfy the minimum Satoshi value requirement.
  - **MIN-SAT-UTXO**: A utxo 𝑢 in a block’s utxo set does not satisfy the minimum Satoshi value requirement.
- Network ID
  - **OUTPUT-NETWORK-UTXO**: A utxo 𝑢 address has the wrong network id.
  - **OUTPUT-NETWORK-TX**: An output of a transaction 𝑡 has the wrong network id.
  - **TRANSACTION-NETWORK**: A transaction 𝑡 has the wrong network id.
- Reference Inputs
  - **NO-REFERENCE-INPUT**: A transaction 𝑡 attempted to reference the input 𝑖 that does not exist or was spent in a previous block.
  - **REFERENCE-INPUT-NO-IDX**: A transaction 𝑡 attempted to spend an input 𝑖 that was not produced by the transaction matching
    the tx hash of 𝑖.

## Archivists

Sundial's archive nodes will store the block data for confirmed blocks. Security properties are easier to achieve for this historical data because, at all times, Sundial's confirmed state utxo contains a chained header hash that pins the entire historical chain of confirmed blocks. There can be no disagreement about different versions of this data—an archive node either stores data that corresponds to the confirmed state header hash, or it does not.

Anyone can choose to run an archive node; however we expect most L2 operators will run archive nodes as this will allow them to more quickly construct blocks with transactions that depend on historical data, we also expect most subscription-service indexers to run their own archive node, as this will allow them to more quickly and reliably provide access to historical data to their users.

As such, an Archive Node implementation will be provided as part of this toolkit and part of the Operator, ready to either be deployed as-is, or used as a reference for building a custom implementation.

### Archive API

Archive nodes are very simple in terms of interaction - they provide a straightforward API for accessing historical blocks - both header & body, and leave the interpretation of that data to the data layer (indexers/wallets/canaries).

- `GET /block/{blockHeight}`: Retrieve the full block (header + body) at the specified height.
- `GET /block/{blockHeight}/header`: Retrieve only the block header at the specified height.
- `GET /block/{blockHeight}/body`: Retrieve only the block body at the specified height.
- `GET /blocks?from={startHeight}&to={endHeight}`: Retrieve a range of blocks between the specified heights.

- `GET /block/{blockhash}`: Retrieve the full block (header + body) with the specified hash.
- `GET /block/{blockhash}/header`: Retrieve only the block header with the specified hash.
- `GET /block/{blockhash}/body`: Retrieve only the block body with the specified hash.

## Facilitators

Facilitators are responsible for assisting users with accelerated deposits/withdrawals. They act as voluntary intermediaries, helping to coordinate transactions more swiftly in return for a fee.

To do this, they first register on a facilitator registry, where users can view available facilitators and the fees they charge. Once a facilitator is selected, they coordinate the deposit/withdrawal process by releasing funds on the target chain from their own liquidity pool, and then waiting to receive the funds from the withdrawal submitted by the user.

Facilitators may reference withdrawal transactions they facilitated to contribute to a verifiable onchain record included in the facilitator registry.

This toolkit will include a simple set of offchain endpoints for interacting with the settlement market as a facilitator - registering & referencing withdrawals.

```Typescript
export const FacilitatorEndpoints = S.Struct({
  register: S.Function(
    S.Struct({
      fee: S.Number,
      receiptAddress: S.L1Address | S.L2Address,
      lpAddress: S.L1Address | S.L2Address,
      credential: S.String
    }),
    S.Void
  ),
  unregister: S.Function(
    S.Struct({
      receiptAddress: S.L1Address | S.L2Address,
      credential: S.String
    }),
    S.Void
  ),
  updateRegistration: S.Function(
    S.Struct({
      fee: S.Number,
      receiptAddress: S.L1Address | S.L2Address,
      lpAddress: S.L1Address | S.L2Address,
      credential: S.String
    }),
    S.Void
  ),
  referenceWithdrawals: S.Function(
    S.Array(
      S.Struct({
        transactionHash: S.String,
        value: S.Number,
        endUser: S.L1Address | S.L2Address
      }),
      S.Void
    ),
  ),
});

```

A separate Sundial Protocol Improvement Proposal will also provide an opt-in standard for indicating further information about withdrawal statistics to offchain observers, built in collaboration with early adopters.

Finally the toolkit will include a ready-to-deploy reference implementation that demonstrates how facilitators can interact with these endpoints.

## Canaries

Canaries have a broader role in monitoring the L2 for any issues, including fraud and other anomalies. They're an opt-in service for users who want additional security and peace of mind.

This is a job that can be accomplished through a variety of methods, so the toolkit will provide a series of exposed APIs in other parts of the protocol that can be easily consumed to build a custom canary service.

### Observation APIs

- **Archive Node API**: Canaries can use the Archive API to access historical block data for analysis.
- **Indexers**: Canaries can use Indexers' provided APIs to monitor the state queue, settlement queue, DAL and scheduler for new blocks and transactions to analyze them in real-time.
- **Layer Node API**: Canaries can interact with the Layer Node directly to access the internal state and monitor for any discrepancies or anomalies.
- **Custom APIs**: Canaries can also work with other network participants to implement their own APIs to gather specific data or insights tailored to their needs.

We are working with partners to develop the first canaries on Sundial, and will expose these APIs as part of the toolkit once they're ready.
