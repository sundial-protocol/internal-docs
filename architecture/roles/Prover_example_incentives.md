# Provers (L1-SL3)

## Description
Provers are responsible for verifying the state of the blockchain and ensuring that transactions are valid. They play a crucial role in maintaining the integrity of the network by monitoring for discrepancies or malicious activity.  

When a block is produced, provers verify the transactions within that block and ensure they adhere to the protocol's rules. If they detect any issues, they can submit a proof to the network and earn the block producer's deposit as a reward.

## Incentives
For a prover to operate effectively within Sundial, it must be incentivised to check for faults consistently. This means its expected rewards from validating transactions must outweigh the costs of performing these checks.  

To ensure this, Sundial will require a significant deposit as collateral for the block producer role. If a prover proves a fault, they can claim this deposit as a bounty. However, because block production is deterministic, faults are expected to be rare.  

To address this, Sundial will implement a **Proof of Diligence (PoD)** system. Under this model:  
- Provers receive **small but consistent rewards** for verifying proposed blocks.  
- Their ongoing costs are covered even when no faults are found.  
- The system ensures a sustainable incentive structure while maintaining strong guarantees of network integrity.  

## Equations
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
