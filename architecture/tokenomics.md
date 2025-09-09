# **Sundial Tokenomics**

## **1\. Core Principle**

Sundial is the first optimistic rollup on Cardano built to function as Bitcoin’s utility and yield layer. Unlike most Layer-2 networks, Sundial introduces no new token - Bitcoin itself is the native asset. Both in its native and wrapped forms, BTC underpins every aspect of the protocol: transaction fees, operator bonds, rewards, and governance.

By anchoring directly to Bitcoin’s liquidity and monetary policy, Sundial achieves economic alignment with the world’s strongest digital asset while avoiding the dilution and distortions that come with inflationary L2 tokens.

## **2\. Role of BTC in Sundial**

### Gas & Fees

Sundial’s native token is wrapped BTC (wBTC), which serves as the universal unit of account for the network. All Layer-2 transactions are ultimately settled in BTC terms, ensuring that the cost of using Sundial is always anchored to Bitcoin’s value.

To maximize flexibility, Sundial employs Babel fees, a mechanism that allows users to pay transaction costs in virtually any bridged asset - not just BTC-based representations like tBTC, iBTC, or cbBTC, but also non-BTC tokens such as ADA. When a transaction is submitted, the chosen fee token is automatically converted into wBTC at the protocol level. This conversion is facilitated by integrated liquidity routes within Sundial, meaning that from the user’s perspective, any supported token can be used to pay gas, while the system consistently settles in wBTC.

This design offers two core benefits:

1. **BTC-Denominated Fee Market**  
   By converging all payments into wBTC, Sundial preserves a unified fee market priced directly in Bitcoin, avoiding the fragmentation that arises in ecosystems with multiple competing fee tokens.

2. **User Accessibility**  
   Users are not required to first acquire wBTC before transacting. They can participate using whichever asset they already hold while Sundial seamlessly normalizes fees into the wBTC standard.

The result is a streamlined, Bitcoin-native fee model where any supported asset can fuel transactions, but the protocol ensures economic neutrality and consistency by settling exclusively in wBTC.

### Network Actors & Incentives

To secure the network, rollup operators must post collateral as a bond, ensuring honest block production. Collateral can be provided in wBTC or any supported token, with Sundial’s Babel fee mechanism automatically converting all deposits into wBTC for final settlement. This gives operators flexibility in sourcing collateral while maintaining a consistent, Bitcoin-native security model anchored in wBTC.

Bond Requirements & Slashing:

* All operator collateral is normalized into wBTC through Sundial’s integrated liquidity routes.

* If an operator is proven to have submitted invalid blocks via fraud proofs, their bond is slashed.

Fraud-Proof Incentives:

* Watchers stake a small challenge bond in wBTC when submitting a fraud proof.

* If the challenge succeeds, the operator’s bond is slashed and the watcher is rewarded in wBTC.

* If the challenge fails, the watcher’s bond is forfeited, deterring spam and griefing attempts.

By requiring all collateral and rewards to settle in wBTC, Sundial ensures that operator security and fraud-proof incentives remain tied directly to Bitcoin’s value, while still allowing participants to enter the system with whichever supported assets they hold.

### Governance

wBTC holders govern Sundial’s core protocol parameters - such as operator bond size, fee splits, and challenge window duration.

Governance is BTC-native, following a “one satoshi, one vote” model. This ensures that every BTC holder, regardless of size, can participate in decision-making, rather than concentrating control only on the largest holders.

By aligning governance power with the smallest indivisible unit of Bitcoin, Sundial enables broad participation while still ensuring that those who provide meaningful BTC liquidity have strong incentives to safeguard and shape the protocol’s evolution.

### Multi-Chain BTC Integration

To maximize liquidity and reduce the fragmentation of Bitcoin representations across ecosystems, Sundial will integrate only secure, decentralized, and proven bridges to major blockchains where wrapped or synthetic BTC exists.

Sundial governance determines which BTC representations are supported. Admission is subject to strict requirements:

* Only tokens that are fully pegged to native BTC with robust economic guarantees and ample liquidity are approved.

* Every supported BTC asset can be seamlessly converted into wBTC for settlement within Sundial.

* Users retain the ability to ramp off at BTC-like value, redeeming their holdings back into the equivalent of native BTC or the desired representation on their preferred blockchain.

By converging approved BTC standards into a single execution environment on Cardano’s optimistic rollup, Sundial unifies liquidity that is currently fragmented across chains. Users can bring whichever supported BTC form they prefer - whether custodial (wBTC), decentralized (tBTC, iBTC), or natively bridged BTC - and interact in one cohesive Layer-2 economy.

## **3\. Treasury & Reserves**

The Sundial treasury is fully denominated in Bitcoin and its wrapped equivalents, making it one of the few Layer-2 protocols whose reserves directly inherit the monetary properties of BTC. All protocol revenues, fees, and slashed collateral accumulate in BTC, creating a continuously growing reserve base that underwrites network security and liquidity.

### **Genesis Reserve Formation**

At launch, Sundial will establish a Genesis Reserve, where a defined tranche of native BTC will be bridged into the protocol to act as its initial economic foundation. This reserve serves several critical purposes:

1. **Security Backbone:** Provides immediate BTC liquidity to seed operator bonds, ensuring that the first cohort of operators can participate without facing prohibitive capital costs.

2. **Fraud-Proof Incentive Pool:** Allocates a portion of genesis reserves to reward early watchers and challengers, guaranteeing that fraud detection is economically viable from day one.

3. **Liquidity Bootstrap:** Funds early liquidity pools, enabling smooth conversion between supported BTC representations (wrapped and native BTC) and reducing slippage for DeFi users.

4. **Insurance Layer:** Creates a capital buffer to cover potential bridge risks or unforeseen operational incidents during Sundial’s bootstrapping phase.

### **Ongoing Reserve Growth**

Unlike inflationary token models, Sundial’s treasury does not rely on minting or emissions. Its reserves grow only with real network usage and real BTC value flows, ensuring that Sundial’s long-term sustainability is strictly aligned with adoption and utility. The genesis reserve provides the initial spark, while protocol revenue and security economics sustain growth thereafter.

After Genesis, the Sundial treasury expands organically through several sustainable mechanisms:

* **Protocol Fees** – A portion of transaction fees and bridge fees, denominated in BTC, flows directly into treasury reserves.

* **Slashed Bonds** – When dishonest operators are penalized, their collateral is slashed. After rewarding successful challengers, the remainder is directed to the treasury.

* **Yield Strategies** – Treasury reserves are deployed into carefully selected, protocol-safe yield opportunities, such as staking and restaking BTC with vetted partners. All rewards, regardless of their source, are denominated in BTC. Even when restaking platforms distribute protocol-specific tokens, these rewards are automatically converted into wBTC through Sundial’s Babel fee mechanism, eliminating reliance on any non-Bitcoin asset. This design ensures that Sundial remains the only restaking platform operating fully under BTC security guarantees, with treasury growth compounding in a sustainable and non-inflationary way - always anchored to Bitcoin’s value.