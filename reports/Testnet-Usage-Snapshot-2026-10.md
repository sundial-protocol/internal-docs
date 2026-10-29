# Testnet Usage Snapshot, October 2026

**Project:** Catalyst project 1300118, Sundial
**Milestone:** Final Milestone, Marketing and Testnet: Prep for Launch
**Compiled:** 2026-10-29

Reviewers of the final milestone asked us to publish the testnet's own numbers: wallets, transactions, bridge activity and success rates, and the output of our system monitoring. As one reviewer put it, small numbers still count as evidence of real usage and stability. This snapshot publishes them. Every figure is read directly from the live testnet node, its public metrics, a public chain, our analytics or our tester survey, on the date given.

## 1. Highlights

| | |
| --- | --- |
| **Tester feedback** | 22 testers rated the public testnet **4.55 / 5**, reported **no critical bugs**, and rated their likelihood of using Sundial on mainnet **4.64 / 5** |
| **L2 activity** | 10 L2 blocks submitted to Cardano preprod, 8 confirmed L2 transactions, 6 deposits from Cardano into the L2, and 9 distinct L2 addresses |
| **Success rate** | 0 rejected transaction submissions and 0 failed block commitments |
| **Stability** | The L2 node answered 100% of 276,479 monitoring checks over the last 48 days |
| **Bridge** | Bitcoin testnet4 to Cardano preprod transfer, confirmed on both chains and independently re-verified |
| **Public monitoring** | Live and public at [dashboard.testnet.sundialprotocol.com](https://dashboard.testnet.sundialprotocol.com/d/ee75b6b8-f1c6-4ef1-9d39-fe50cc55a274/midgard) |
| **Public interest** | 80 website visitors and 14 documentation visitors in the first three days of web analytics, with the testnet faucet the second most visited page |

## 2. Testnet usage

### Sundial L2

Source: the testnet node's database, read 2026-10-28, covering the current deployment from 2026-07-30.

| Measure | Count | Dates |
| --- | --- | --- |
| Distinct L2 addresses (wallets) | 9 | |
| Faucet claims | 8, from 6 addresses | 2026-07-30 to 2026-09-30 |
| Deposits from Cardano preprod into the L2 | 6 | 2026-07-30 to 2026-09-11 |
| Confirmed L2 transactions | 8 | 2026-08-12 to 2026-09-30 |
| L2 blocks built and submitted to Cardano preprod | 10 | 2026-07-30 to 2026-09-30 |

### Bitcoin staking (Dawn and the Yield Marketplace)

Source: the yield backend's database and public API, read 2026-10-28 and 2026-10-29.

| Measure | Count |
| --- | --- |
| Yield programs listed in the marketplace | 6 |
| Deposit intents created | 6, from 2 Bitcoin addresses, 2026-05-06 to 2026-05-23 |

### Bridge

The Bitcoin to Cardano transfer documented in [M5.2](M5.2-Transactions-Report.md) is confirmed on both chains and can be checked on public explorers:

| Step | Network | Transaction |
| --- | --- | --- |
| Charm minted | Bitcoin testnet4 | [`3be09f7f…93d4cc`](https://mempool.space/testnet4/tx/3be09f7f1539f3a62562dceea80278e28b57882b8803a2418862f2a2c693d4cc) |
| Destination prepared | Cardano preprod | [`8c287322…fd9e55`](https://preprod.cardanoscan.io/transaction/8c287322a56b066a0e6eca33f6376483254ec51e41c6f9f208fb314135fd9e55) |
| Charm beamed to Cardano | Bitcoin testnet4 | [`53245515…7ea162`](https://mempool.space/testnet4/tx/532455151c9127f4c5b5411ae8dcffa8ccde1fb2ce9ddda0300c38c6067ea162) |

## 3. Success rate and stability

Source: the node's public metrics, read 2026-10-29. The node's current run began on 2026-09-11.

| Measure | Result |
| --- | --- |
| Rejected transaction submissions | 0 |
| Failed block commitments | 0 |
| Monitoring checks the node answered | 276,479 of 276,479 over 48 days (100%) |

All of these are charted on the [public monitoring dashboard](https://dashboard.testnet.sundialprotocol.com/d/ee75b6b8-f1c6-4ef1-9d39-fe50cc55a274/midgard): transactions received and queued, blocks built, submitted and merged, transactions per block, rejected submissions and commitment failures, and container CPU and memory. Anyone can open it without an account.

## 4. Tester feedback

### Final milestone testnet survey

[Raw responses, public and unedited](https://docs.google.com/spreadsheets/d/1ec9JWN-vXE2xwr0tztLsjtpTg35mHmEYdourvxPzuZQ/edit). 22 responses, 10 to 25 September 2026.

| Question | Result |
| --- | --- |
| Overall experience | **4.55 / 5** (12 rated 5, 10 rated 4) |
| Critical bugs or errors encountered | **None**, in all 22 responses |
| Likelihood of using Sundial on mainnet | **4.64 / 5** (14 rated 5, 8 rated 4) |
| Wallet connectivity | 17 Excellent, 5 Good |
| Transaction latency | 13 Excellent, 9 Good |
| Dashboard responsiveness | 13 Excellent, 9 Good |
| Staking interface | 10 Excellent, 11 Good, 1 Average |

Testers' suggestions centred on the dashboard: clearer staking confirmations, a more prominent pending-transaction status, recent activity and history on the dashboard, tooltips explaining figures, and a clearer wallet-connection state.

### From feedback to code

Tester feedback goes straight into the product. The [Milestone 5 beta](M5.7-User-Feedback-Report.md) (23 testers, rated 4.39 / 5) led to [sundial-web#38, "Ux feedback"](https://github.com/sundial-protocol/sundial-web/pull/38), merged on 2026-07-13: it surfaced clearer deposit errors and explained when the Sign & Deposit action is unavailable, fixed amount precision, and restored dashboard navigation. The changes requested in the final milestone survey are in development on the dashboard now.

## 5. Public interest

Source: PostHog, from the launch of web analytics on 2026-10-26 to 2026-10-29.

| Site | Visitors | Page views |
| --- | --- | --- |
| [sundialprotocol.com](https://www.sundialprotocol.com) | 80 | 94 |
| [docs.sundialprotocol.com](https://docs.sundialprotocol.com) | 14 | 18 |

| Most visited website pages | Visitors |
| --- | --- |
| Home | 15 |
| Testnet faucet | 14 |
| News: Unlocking DeFi on Sundial | 9 |
| News: Sundial on YouTube | 8 |
| Sundial L2 | 7 |

The testnet faucet drew nearly as many visitors as the home page, a direct signal of interest in trying the testnet.

## 6. Verify it yourself

| What | How |
| --- | --- |
| L2 transactions | `GET https://rpc.testnet.sundialprotocol.com/tx?tx_hash=<hash>` returns the full transaction. Examples: `9ad083dcc3751ef2a50b3030261c6fcfb499bd2a00f55154dde28971a0f045c0`, `5dba244aef0a3327b463c541bb1ea0d199bfa4437a22ce29db47fde42ee10391`, `eb40f88e892c1ca9aa0b4ae6dc8c73bd2825ca0bc271d6921e03c5c0f45106c3` |
| Bridge transactions | The explorer links in section 2 |
| Live metrics | The [public monitoring dashboard](https://dashboard.testnet.sundialprotocol.com/d/ee75b6b8-f1c6-4ef1-9d39-fe50cc55a274/midgard) |
| Tester responses | The [raw survey sheet](https://docs.google.com/spreadsheets/d/1ec9JWN-vXE2xwr0tztLsjtpTg35mHmEYdourvxPzuZQ/edit) |
| Try the testnet | The [testnet faucet](https://www.sundialprotocol.com/testnet/faucet) |

## 7. What's next

Each item below follows a reviewer recommendation or adds a measure to the next snapshot.

| Improvement | What it adds |
| --- | --- |
| **Dashboard analytics** | Connected wallets and a full funnel from visit to connected wallet to confirmed deposit, published in each snapshot |
| **Faucet analytics** | Claims tracked end to end, from page visit to funded address |
| **Bitcoin deposit indexing on testnet4** | Confirmed Dawn deposits, deposit volume and depositor counts, on the same network as the bridge |
| **Expanded L1 data capacity for the L2 node** | Continuous, around-the-clock L1 synchronisation as testnet traffic grows |
| **Long-term testnet history** | Testnet activity preserved across upgrades, so every snapshot builds on the last |
| **Sundial's own state-queue contract on testnet** | L2 commitments countable by anyone on a Cardano explorer |
| **Feedback-tagged changes** | Every change that comes from tester feedback says so in its pull request, so the path from community input to code is visible |
| **Survey dashboard improvements** | The final milestone survey's requests shipped to the testnet dashboard |
| **Public answers to community questions** | Requests for metrics on social channels answered with a link to this snapshot and the live dashboard |
