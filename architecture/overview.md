# Architectural Overview

![User Flows & Services](services.png)

The architecture of Sundial is divided into 7 layers, the first 2 of which are the Layer 1 (Cardano) and Layer 2 (Sundial) protocols. The remaining layers are built to enable ease of use and integration with the protocol, as well as to provide a user-friendly experience.

This document provides an overview of the architecture, including the key services and their interactions. The architecture is designed to be modular and extensible, allowing for future enhancements and integrations.

## L1: Cardano

Sundial settles to the Cardano blockchain, which provides the foundational layer for Sundial. Cardano's track record for security and stability is rivaled only by Bitcoin, and its proof-of-stake consensus mechanism ensures that it can scale to meet the needs of a growing user base without compromising on security or decentralization. The fully deterministic nature of Cardano's smart contracts also makes it an ideal choice for building a high-confidence optimistic Layer 2 protocol.

Cardano hosts a suite of smart contracts that Sundial uses to manage its operations. These include a state queue, settlement queue, scheduler, deposit contract, an operator registry, and a suite of contracts for proving fraud on the L2 in a fully deterministic manner.

Cardano is also used as a Data Availability (DA) layer, meaning it stores the data needed for Sundial to operate. As an optimistic rollup, Sundial only publishes block headers into the state queue. The DA layer ensures that block data is secure and can be accessed by anyone who needs it for fraud proving or other onchain purposes.

## L2: Sundial

Layer 2 is the Sundial protocol itself, which is built on top of the Cardano blockchain. It provides the core functionality for the protocol, including the state queue, settlement queue, and scheduler.

Sundial is built with the Midgard L2 Framework, which is a set of tools and libraries that make it easier to build Layer 2 protocols on Cardano. It provides a set of abstractions for managing state, transactions, and other core functionality. Included in this layer are the services mentioned above, as well as the following:

- **Data Archive**: A decentralized, data-duplicated service that stores historical data for the protocol, allowing users to access past transactions and states.
- **Data Availability (DA) Layer**: This layer ensures that the data needed for Sundial to operate is securely stored and accessible. It is implemented on Cardano, leveraging its Leios blob storage for easy accessibility onchain without undue cost or bloat to the L1.

The Layer 2 also makes use of the Plutus VM, allowing for isomorphism between L1 & L2 smart contracts. This allows dapps to easily interact with both layers, and for developers to build complex applications that leverage the strengths of both Cardano and Sundial. The transactional determinism of the Plutus VM also allows for greater security and trustlessness, as developers can easily verify the correctness of their applications with formal verification methods.

## SL3: Data Layer

The Data Layer has 2 types of services:

- **Indexers** provide a way to index and query data from the protocol, allowing users to access historical data and perform analytics. They also should provide a way to view the upcoming block producers, enabling wallets to intelligently broadcast their transactions to the relevant operators. (Assuming a VRF rotation instead of round-robin.)
- **Security Services** provide a way to ensure the security of the protocol, including fraud detection and prevention mechanisms. These services are designed to be modular and extensible, allowing for future enhancements and integrations.

## SL4: Construction Layer

The Construction Layer provides a set of tools and libraries for building applications on top of the Sundial protocol. It includes the following services:

- **Offchain Specifications**: A set of specifications that define how applications can construct transactions and interact with the protocol. These are backwards compatible with Cardano's CIP-30 wallet standard, allowing for easy integration with existing wallets and applications.
- **Wallet Specifications**: A set of specifications that define how wallets can interact with the protocol, including transaction construction and signing. This includes specifications for interacting with both the Bitcoin and Cardano networks, as well as the Sundial L2.
- **Shared Liquidity Pools**: A set of liquidity pools made available to dapps that pass our verification process, as well as the native dapps integrated into the yield platform. The initial design includes pools for stablecoins, wrapped Bitcoin, and native Ada.

## SL5: Function Layer

The Function Layer includes a suite of commonly desired use cases that are made available to users as an inherent part of the protocol. They include:

- **Bridges** between Bitcoin and Cardano: These bridges allow users to move assets between the two networks, enabling cross-chain transactions and interactions. We're working alongside 3 teams (Charms, IOG, Fluidtokens) that are developing these zero-trust bridges, and have concrete plans for integration with them via our bridge operator tooling.
- **Yield Instruments**: These allow users to earn yield on their assets, providing a way to generate passive income. The yield instruments are designed to be secure and easy to use, allowing users to earn yield without having to manage complex strategies. They allow users to choose where their yield is generated by selecting from a range of options, to tailor their risk and return profiles.
- **Watchers**: These are services that monitor the protocol for specific events or conditions, such as fraud detection or accelerated deposits/withdrawals. They operate as independent actors, and incentivized to provide accurate and timely information to users.

## SL6: Routing Layer

The Routing Layer provides a set of smart services that interprets and prepares user inputs for interaction in the Function Layer. This includes:

- **Bridging Services**: These services facilitate user interactions with the bridges, allowing them to move assets between Bitcoin and Cardano networks seamlessly. They also monitor the status of the bridges and provide users with real-time information about their transactions.
- **Staking Services**: These allow users to stake their assets in the protocol, earning rewards for participating in the network. They are designed to be easy to use and secure, allowing users to stake their assets without having to manage complex strategies.
- **Facilitator Registry**: This provides a way for facilitators to register their services, allowing users to discover and interact with them easily. It includes information about each facilitator's fees, supported networks, and other relevant details.

## SL7: User Layer

The User Layer is the topmost layer of the architecture, providing a user-friendly interface for interacting with the protocol. It includes:

- **User Interfaces**: These are the front-end applications that users interact with, including web and mobile applications. They are designed to be intuitive and easy to use, allowing users to interact with the protocol without needing to understand the underlying complexities.
- **User Actions API**: This API provides a way for users to interact with the services in SL6 and SL5 programmatically. It is designed to be easy to use and secure, allowing developers to build applications without having to manage complex interactions with the protocol.
- **Watcher Toolkit**: This provides users with the tools they need to set up and manage their own watcher services. It includes documentation, containerized applications, and other resources to help new watchers get started quickly and easily.
- **Operator Toolkit**: This provides users with the tools they need to set up and manage their own operator services. It includes documentation, a CLI, and other resources to help operators tailor the toolkit to their specific requirements.
