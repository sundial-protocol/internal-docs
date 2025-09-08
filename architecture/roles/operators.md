# Operators

Operators are the backbone of the Sundial protocol, responsible for maintaining the network's integrity and performance. They play a crucial role in ensuring that transactions are processed efficiently and securely.

## Layer Operators (Block Producers)

### Responsibilities

Operators have several key responsibilities, including:

- **Transaction Processing**: Operators are responsible for processing transactions on the network, ensuring that they are valid and adhere to the protocol's rules.
- **Network Maintenance**: Operators play a crucial role in maintaining the health and performance of the network, monitoring for any issues or anomalies that may arise.
- **Incentive Alignment**: Operators are incentivized to act in the best interests of the network, with rewards tied to their performance and the overall health of the ecosystem.

### Incentivization

To become an operator on the Sundial network, participants must stake a certain amount of capital as collateral, aligning their interests with the rules of the protocol.

This collateral serves as a guarantee that operators will act in accordance with the protocol's rules and will be penalized for any malicious behavior. Should an operator attempt to act against the protocol's rules, they risk losing their staked collateral to any single alert watcher without making any permanent changes to the ledger.

Because of the deterministic nature of the protocol, the malicious operator takes on all risk in such a scenario, and the contesting watcher takes on no risk at all.

In addition, we include a service that monitors operator performance and provides feedback on their activities. This helps to ensure that operators are held accountable for their actions and that they are incentivized to act in the best interests of the network.

If the operators fail to meet the required standards, they may be penalized by being temporarily suspended from the network. This helps to maintain a high level of performance and reliability within the operator community.

### Tooling

To support operators in their roles, we provide a comprehensive toolkit. Information on the operator toolkit can be found [here](../layers/SL7/operator-toolkit.md).

## Bridge Operators

Much of what bridge operators will do is still TBD as we work with different bridging solutions. More information will come as we finalize our approach.

At present we are working with 3 different bridging solutions, with the expectation that we will select one or more of them to integrate into the protocol. These include:

1. Cardinal
2. Charms
3. Unnamed Bridge by FluidTokens

Expected requirements for operators will change slightly based on the bridging solution(s) selected.

### Cardinal Operator Requirements

Cardinal’s operator model is a balance of **trust minimization, cryptographic assurance, and resilient infrastructure**. By relying on **MuSig2**, **HTLC**, and **BitVMX-based fraud proofs**, it achieves cross-chain interoperability without custodial risk.

Operators are expected to uphold security, availability, and correctness under adversarial conditions—all grounded in the 1-of-n honest assumption.

Cardinal operators are responsible for monitoring the bridge's performance, detecting any anomalies or issues, and taking appropriate action to resolve them. This may include coordinating with other operators, communicating with users, and implementing technical fixes as needed.

#### Availability & Infrastructure Requirements

- Operators must maintain **high uptime** and resilient infrastructure to meet availability expectations—capable of handling network-level disruptions or denial-of-service attempts.
- Their infrastructure must be robust enough to keep the protocol active despite adversarial conditions.

#### Custody, MuSig2, and HTLC Security Mechanisms

- Wrapped Bitcoin assets (including Ordinals) are **locked on-chain** using **MuSig2 aggregated multi-signature schemes**, managed by the operator set.
- An **HTLC (Hashed Timelock Contract)** enforces redemption conditions and time-bound recovery mechanisms for pegged assets.
- Operators must correctly implement and coordinate **MuSig2 multi-signature logic and HTLC conditions** to ensure trust-minimized custody and secure asset locking.

#### Bridging Flow: Lock, Mint, Burn, Release

Operators manage the cross-chain asset lifecycle:

1. **Locking**: A Bitcoin UTxO (e.g., an Ordinal) is locked via the MuSig2-enabled mechanism.
2. **Verification & Minting**: Operators observe blockchain events and, upon validating the lock, trigger minting of a 1:1 pegged Sundial asset (e.g., cNFT) using eUTxO smart contracts.
3. **Burn / Peg-Out**: When the user burns the wrapped asset on Sundial, operators oversee the corresponding Bitcoin UTxO’s release, ensuring integrity in cross-chain state transitions.

This lifecycle must ensure **atomicity** and **reversibility** of asset transfers, grounded in a fraud-proof-backed trust model.

#### Fraud-Proofs via BitVMX Off-Chain Verification

- The **BitVMX framework** is central to operator responsibilities: it provides **off-chain execution and optimistic verification**, with on-chain dispute resolution if misbehavior is detected.
- Operators must be prepared to engage in BitVMX's **Dispute Resolution Protocol (DRP)**—a verification game where challengers isolate incorrect computation steps through cryptographic trace dissection.
- This ensures a **trust-minimized system**: only a single honest operator is required to enforce correctness via fraud proofs.

#### At a Glance

| Requirement                      | Expectation from Cardinal Operators                          |
| -------------------------------- | ------------------------------------------------------------ |
| **Honesty**                      | At least one honest operator required (1-of-n model)         |
| **Availability**                 | High uptime, resilience to network attacks                   |
| **MuSig2 / HTLC Implementation** | Secure multi-sig and time-locked contracts on Bitcoin        |
| **Cross-Chain Lifecycle**        | Correct lock, mint, burn, and release phases                 |
| **BitVMX Dispute Handling**      | Ability to challenge and respond in fraud-proof DRP          |
| **Censorship & Fee Strategy**    | Adaptive fee bidding; resilience to censorship attempts      |
| **Finality Consideration**       | Wait for confirmations per-chain to avoid double-spend/forks |
| **Extensible Architecture**      | Support for future types of UTxOs beyond Ordinals            |

#### Further Reading

- ["Cardinal Whitepaper"](https://8848114.fs1.hubspotusercontent-na1.net/hubfs/8848114/Bitcoin-Cardano%20BitVMX%20Ordinals%20Wrapping.pdf)
- ["Cardinal Spec"](https://github.com/input-output-hk/cardinal-spec?utm_source=chatgpt.com)
- ["Introducing Cardinal"](https://www.fairgate.io/post/15-introducing-cardinal-how-bitvmx-bridges-bitcoin-ordinals-to-cardano-unlocking-cross-chain-defi)
- ["BitVMX | Bitcoin Computation"](https://bitvmx.org/)

### Charms

Charms enables **programmable, portable assets** on Bitcoin (and other UTXO-based chains) without bridges or custodians. This is achieved via **client-side validation** powered by **recursive zkVM (Groth16) proofs**. These proofs validate “spells” (metadata that “enchants” Bitcoin UTXOs) and can travel—or _beam_—across chains while maintaining their logic and provenance.

- Assets (charms) and app logic are verified entirely by clients (e.g., wallets), not by on-chain logic.
- Beaming involves using **Merkle proofs coupled with proof-of-work confirmations** from source chain (Bitcoin) blocks to prove inclusion and chain stability.

---

## Operator Role in Charms: Bitcoin Oracle Operator

### Technical Overview

These operators focus narrowly on one function: **attesting to Bitcoin block inclusion and confirmations**, providing the data necessary for clients to produce or verify **spell proofs** in cross-chain operations like beaming.

#### Bitcoin Node Maintenance

- **Run a full, fully validating Bitcoin node** (e.g., Bitcoin Core), monitoring the blockchain to obtain:

  - Block headers (and optionally full block data).
  - Chain height and block canonicality.
  - Transaction inclusion data and Merkle proofs.

#### Oracle Attestation Service

- Extract relevant data:

  - **Merkle inclusion proofs** of beaming transactions.
  - **Proof-of-work-based finality**, confirming a block resides on the main chain with sufficient confirmations.

- Optionally, package:

  - Block header hashes.
  - Proof-of-work depth (e.g., confirmation count).
  - Merkle proofs for beamed transactions.

#### Integration with Charms’ Beaming Protocol

The Charms protocol requires:

- Verification that charmed outputs (beamed) appear in the blockchain’s main branch.
- Provision of:

  - **Merkle proof** for inclusion.
  - **Chain stability proof**, e.g., X blocks of proof-of-work on top of that block.

Operators feed this information to:

- **Charms client libraries**, enabling them to verify beaming transactions within recursive zkVM proofs.
- Possibly, other aggregators or systems that support cross-chain validation.

#### Operational Requirements & Expectations

| Requirement                | Expectation                                                                                                     |
| -------------------------- | --------------------------------------------------------------------------------------------------------------- |
| **High availability**      | Oracle must be online and responsive; uptime ensures beaming is timely.                                         |
| **Accuracy & correctness** | Must reflect Bitcoin consensus accurately. Confidentiality-of-finality and reorg handling are key.              |
| **Reorg detection**        | Capability to detect and update attestations promptly in case of chain reorganizations.                         |
| **Deterministic data**     | Ensure oracle data (Merkle and PoW proofs) strictly corresponds to canonical blocks.                            |
| **Optional signatures**    | For higher accountability, oracle messages can be signed so clients can verify origin authenticity.             |
| **Redundancy**             | Multi-operator setups reduce risk: if one operator malfunctions, others can ensure accurate data flow.          |
| **Security hygiene**       | Node must be secure; connections must resist eclipse or partition attacks that could compromise data integrity. |

#### Minimal Technical Stack

- **Bitcoin node** (e.g., Bitcoin Core).
- Lightweight **oracle service** that:

  - Extracts headers, confirmations, Merkle proofs.
  - Packages attestations in client-friendly format.
  - Optionally signs the data for integrity.

- **Monitoring systems** to ensure uptime and detect fork events.

---

#### Operator Focus & Deliverables

- **What they don’t do**: They’re not custody agents, not zkVM proof providers, not app contract evaluators.
- **What they do**:

  1. Track the Bitcoin blockchain.
  2. Generate inclusion and PoW-based proofs.
  3. Serve these proofs to Charms’ client ecosphere.
  4. Maintain high reliability and correctness.

This role is fundamental in ensuring **cross-chain beaming integrity**—clients rely on these proofs to validate spells and charms properly when moving assets across chains.

#### Further Reading

[Charms: Programmable Assets on Bitcoin and Beyond](https://charms.dev/Charms-whitepaper.pdf)

### Unnamed Bridge by FluidTokens

The Unnamed Bridge by FluidTokens is an innovative solution aimed at facilitating seamless asset transfers between the Sundial network and other blockchain ecosystems. While specific details are still being finalized, the bridge is expected to incorporate advanced features to enhance security and user experience, and include a 1-of-n or fully-trustless design.

Details on operator requirements and responsibilities will be provided as they become available.
