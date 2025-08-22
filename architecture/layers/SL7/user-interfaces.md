# Sundial User Interfaces

One of the most important parts of providing a good user experience is the actual interface design. This includes not only the visual aspects but also the overall usability and accessibility of the interfaces.

Interfaces:

- Financial Dashboard
- Chain Explorer
- Network Status Dashboard

These are intended to face end-users, providing them with the tools they need to interact with the protocol easily and intuitively. This includes 3 interfaces. Each one will have a web interface, as well as mobile applications for iOS and Android.

## Financial Dashboard

The financial dashboard is a one-stop shop for users to manage their Bitcoin portfolio in a user-friendly interface. It provides real-time insights into their holdings, transaction history, and market trends, allowing users to make informed decisions about their investments.

Important components include:

- **Portfolio Overview**: A visual representation of the user's portfolio, including asset allocation, amount locked, risk-reward profile, performance metrics, and historical trends.
  - **Transaction History**: A detailed log of all transactions, including deposits, withdrawals, and trades, with filtering and search capabilities.
- **Yield Catalog**: A comprehensive list of available yield-generating opportunities, including staking, lending, and liquidity provision options. All insights should have clear indication of the anticipated risk & reward.
  - **Staking Opportunities**: Information on available staking pools, including APY, lock-up periods, and supported assets.
  - **Lending Opportunities**: Details on lending platforms, including interest rates, collateral requirements, and supported assets.
  - **Liquidity Provision Opportunities**: Information on liquidity pools, including fees, impermanent loss risks, and supported assets.
  - **Alternative Investment Opportunities**: A curated list of alternative investment options, including insurance, real estate, and other digital assets.
  - **Prebuilt Strategies**: A collection of prebuilt strategies for users to easily deploy, including risk profiles and expected returns.
- **Market Insights**: Real-time data on market trends, including price charts, order book depth, and liquidity metrics.
- **Learning Resources**: A set of educational materials and resources to help users understand the financial instruments available and make informed decisions.

## Chain Explorer

The chain explorer provides users with a comprehensive view of the blockchain, allowing them to explore blocks, transactions, and addresses in detail. Users can search for specific transactions or addresses and view their history and associated data.

We will work with existing chain explorers to embed their systems within the Sundial UI, providing users with familiar interfaces for exploring blockchain data and supporting ecosystem adoption.

## Network Status Dashboard

The network status dashboard provides users with real-time information about the health and performance of all components of the Sundial network. It includes key metrics, alerts, and insights to help users understand the current state of the network, split into a collection of views or sub-dashboards.

| View                           | Description                                                                                                                                                                             |
| ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **L1 Dashboard**               | Provides insights into the performance and health of the Cardano & Bitcoin Layer 1 blockchains. Includes block times, transaction throughput, and network latency.                      |
| **L2 Dashboard**               | Offers insights into the performance and health of Layer 2 solutions built on Cardano & Bitcoin. Includes transaction speeds, costs, user adoption, and health of layer-node operators. |
| **Security Services**          | Shows performance and health of public data tooling, specifically monitoring known Canaries.                                                                                            |
| **Liquidity Dashboard**        | Displays liquidity available across Sundial, including pool depths, trading volumes, and slippage rates.                                                                                |
| **Tool Vulnerabilities Board** | Provides insights into the security posture of tools built on Sundial, such as wallets and transaction builders.                                                                        |
| **Bridge Status Dashboard**    | Gives real-time information about the health and performance of bridges connecting Sundial to other blockchains, including trust assumptions, utilization, success rates, and latency.  |
| **Archive Dashboard**          | Shows historical performance and usage patterns, including data availability, blocks stored, user activity, and system performance over time.                                           |
| **Facilitator Registry**       | Lists all active facilitators, including performance metrics, user ratings, and available services. Facilitators can register and update their information.                             |
| **Participant Overview**       | Provides insights into participants within Sundial broken down by role, including user counts and activity levels, to help users understand the ecosystem.                              |
