# Operator Incentives

## Layer Operator Incentives

Layer Operators (Block Producers) are incentivized through a combination of:

- **Staking rewards** for processing transactions and maintaining network health.
- **Collateral requirements**: Operators must stake capital as collateral, which can be slashed for malicious behavior or poor performance.
- **Penalties**: Operators risk losing their collateral and being suspended for protocol violations.

### Equations

The following equation determines the expected payout for a Layer Operator:

$$
\mathbb{E}[Payout_{LO}] =  (\lambda_{tx} \cdot TxFee \cdot \tau_{rotation} / N_{operators}) - (Collateral_{btc} \cdot C_{opportunity}) - C_{operation}
$$

Where:
- $$\lambda_{tx}$$ = Expected number of transactions processed
- $$TxFee$$ = Average gas fee per transaction sent to block
- $$\tau_{rotation}$$ = Operation window for block production
- $$N_{operators}$$ = Number of layer operators actively participating
- $$Collateral_{btc}$$ = Amount of BTC locked to participate
- $$C_{opportunity}$$ = Opportunity cost of locked collateral
- $$C_{operation}$$ = Cost of operating a node


**Condition for sustainability:**

$$
\mathbb{E}[Payout_{LO}] > 0
$$

## Bridge Operator Incentives

We allow for flexibility in the fee structure for bridge operators, as the fees that may be charged by bridge operators changes based on the bridge and type of asset being bridged.

In order to account for this we include the Bridging Service, which helps support users in finding and using the best bridge for their needs.

### Cardinal Operator Incentives

Cardinal Operators are incentivized through:

- **Transaction fees** for facilitating cross-chain operations (lock, mint, burn, release).
- **Adaptive fee bidding**: Operators can adjust fees based on network conditions.
- **Fraud-proof rewards**: Operators may earn additional rewards for successfully participating in BitVMX dispute resolution.
- **Penalties**: Operators who fail to maintain uptime, security, or correctness may lose their role or face penalties.

### Equations

The following equation determines the expected payout for a Cardinal Operator:

$$
\mathbb{E}[Payout_{CO}] = \lambda_{tx} \cdot Fee_{tx} + Reward_{fraudproof} - Penalty_{misbehavior} - C_{operation} - C_{capital}
$$

Where:

- $$\lambda_{tx}$$ = Expected number of cross-chain transactions
- $$Fee_{tx}$$ = Average fee per transaction
- $$Reward_{fraudproof}$$ = Rewards from successful fraud-proof participation
- $$Penalty_{misbehavior}$$ = Penalties for protocol violations
- $$C_{operation}$$ = Cost of operating as a Cardinal Operator
- $$C_{capital}$$ = Cost of required infrastructure

**Condition for sustainability:**

$$
\mathbb{E}[Payout_{CO}] > 0
$$

### Charms Operator Incentives

Charms Operators (Bitcoin Oracle Operators) are incentivized through:

- **Attestation fees** for providing accurate and timely oracle data (block inclusion, confirmations, Merkle proofs).
- **Redundancy rewards**: Multi-operator setups can increase reliability and share fees.
- **Optional signature rewards**: Operators may charge extra for signed attestations.
- **Penalties**: Poor availability, incorrect data, or slow response may result in loss of reputation or removal from the operator set.

### Equations

The following equation determines the expected payout for a Charms Operator:

$$
\mathbb{E}[Payout_{ChO}] = \lambda_{attest} \cdot Fee_{attest} + Reward_{redundancy} + Reward_{signature} - Penalty_{misbehavior} - C_{operation}
$$

Where:

- $$\lambda_{attest}$$ = Expected number of attestations provided
- $$Fee_{attest}$$ = Average fee per attestation
- $$Reward_{redundancy}$$ = Additional rewards for redundancy
- $$Reward_{signature}$$ = Additional rewards for signed attestations
- $$Penalty_{misbehavior}$$ = Penalties for incorrect or late data
- $$C_{operation}$$ = Cost of operating as a Charms Operator

**Condition for sustainability:**

$$
\mathbb{E}[Payout_{ChO}] > 0
$$

# Watcher Incentives

Each of the 4 watcher roles has different incentives and cost structures. Below is an outline of the incentives for each role. The constraints on these incentive systems come from practicalities:

- Each watcher must be sufficiently incentivized to make running the specialized service on top of the core watcher service worthwhile.
- Running all watchers in tandem must be incentivized to sufficiently offset the cost of running them together to make the cost of running the core service worthwhile.

## Prover Incentives

For a prover to operate effectively within Sundial, it must be incentivised to check for faults consistently. This means its expected rewards from validating transactions must outweigh the costs of performing these checks.

To ensure this, Sundial will require a significant deposit as collateral for the block producer role. If a prover proves a fault, they can claim this deposit as a bounty. However, because block production is deterministic, faults are expected to be rare.

To address this, Sundial will implement a **Proof of Diligence (PoD)** system. Under this model:

- Provers receive **small but consistent rewards** for verifying proposed blocks.
- Their ongoing costs are covered even when no faults are found.
- The system ensures a sustainable incentive structure while maintaining strong guarantees of network integrity.

### Equations

The following equation determines the minimum bounties and PoD payments required for provers:

$$
\mathbb{E}[Payout_{Pr}] = (\lambda_{fault} + \pi_{win}) \cdot Bounty + PoD_{Payment} - C_{operation} - C_{capital}
$$

Where:

- $$\lambda_{fault}\$$ = Probability of a fault occurring
- $$\pi_{win}\$$ = Probability of winning the bounty
- $$Bounty\$$ = Total fault bounty (from block producer deposit)
- $$PoD_{Payment}\$$ = Small constant payments from Proof of Diligence
- $$C_{operation}\$$ = Cost of operating a prover
- $$C_{capital}\$$ = Cost of setting up a prover

$$(\lambda_{fault} + \pi_{win}) \cdot Bounty\$$ represents the average earnings from bounties over time.

$$(\ C_{operation} + C_{capital}\) $$ represents total costs

**Condition for sustainability:**

$$
\mathbb{E}[Payout_{Pr}] > 0
$$

Using this framework, we can calculate the required bounty size and the level of regular PoD payments necessary to ensure sufficient provers continuously check transactions.

## Facilitator Incentives

Facilitators are incentivized through small fees charged for each accelerated deposit or withdrawal they process. These transactions are expected to be frequent, but the fees will be small, so facilitators will need to handle a large volume of transactions to make it worthwhile.

### Equations

The following equation determines the minimum fee structure required for facilitators:

$$
\mathbb{E}[Payout_{Fa}] = \lambda_{tx} \cdot (UserFee - TxFee) - C_{operation} - C_{capital}
$$

Where:

- $$\lambda_{tx}$$ = Expected number of transactions processed
- $$UserFee$$ = Average fee charged per transaction
- $$TxFee$$ = Average gas fee per transaction
- $$C_{operation}$$ = Cost of operating a facilitator
- $$C_{capital}$$ = Cost of setting up a facilitator

$$(\lambda_{tx} \cdot UserFee - TxFee) $$ represents the average earnings from facilitation over time.
$$(TxFee + C*{operation} + C*{capital}) $$ represents total operational costs, not including gas fees.

**Condition for sustainability:**

$$
\mathbb{E}[Payout_{Fa}] > 0
$$

## Canary Incentives

Canaries are incentivized through fees charged for their monitoring services. They provide an opt-in service for users who want additional security and peace of mind. Canaries can set their own fees and terms of service, allowing them to tailor their offerings to the needs of their users.

### Equations

The following equation describes the expected minimum fee structure required for canaries. Canaries are free to choose their own monetization strategy as desired.

$$
\mathbb{E}[Payout_{Ca}] = \lambda_{sub} \cdot Fee - C_{operation}
$$

Where:

- $$\lambda_{sub}\$$ = Expected number of subscriptions
- $$Fee\$$ = Average fee charged per subscription
- $$C_{operation}$$ = Cost of operating a canary

$$(\lambda_{sub} \cdot Fee) $$ represents the average earnings from subscription fees over time.

**Condition for sustainability:**

$$
\mathbb{E}[Payout_{Ca}] > 0
$$

## Archivist Incentives

Archivists are incentivized through fees charged for accessing historical data. They ensure that historical data is accurately recorded and accessible, providing a reliable source of information for users and other services. Archivists can charge fees based on the availability of the data and the demand for it.

### Equations

The following equation describes the expected minimum fee structure required for archivists:

$$
\mathbb{E}[Payout_{Ar}] = \lambda_{access} \cdot Fee - C_{operation}
$$

Where:

- $$\lambda_{access}\$$ = Expected number of data access requests
- $$Fee\$$ = Average fee charged per access
- $$C_{operation}\$$ = Cost of operating an archivist
- $$C_{capital}\$$ = Cost of setting up an archivist

$$(\lambda_{access} \cdot Fee) $$ represents the average earnings from access fees over time.
$$(C*{operation} + C*{capital}) $$ represents total costs

**Condition for sustainability:**

$$
\mathbb{E}[Payout_{Ar}] > 0
$$

## Combined Incentives

To ensure that running all watcher roles in tandem is economically viable, the combined expected payouts must exceed the combined costs of operating all services together.

$$
\mathbb{E}[Payout_{Total}] = \mathbb{E}[Payout_{Pr}] + \mathbb{E}[Payout_{Fa}] + \mathbb{E}[Payout_{Ca}] + \mathbb{E}[Payout_{Ar}] - C\_{operation} - C\_{capital}
$$

Where:

- $$C_{operation}$$ = Cost of running the core watcher service
- $$C_{capital}$$ = Cost of capital for the core watcher service

**Condition for overall sustainability:**

$$
\mathbb{E}[Payout_{Total}] > 0
$$
