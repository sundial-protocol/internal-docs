# The Sundial User Action API

A simple REST API for interacting with the Sundial protocol. This API provides endpoints for users & developers to perform various actions, such as submitting Tx requests, bridging, and using staking functionality.

It also includes a few basic informational endpoints to retrieve data used for actions, such as operator & account details, for data consistency purposes.

This will be hosted as a public service, but can also be self-hosted by users who want to run their own instance of the API. All functionality will be accesible elsewhere in decentralized ways (via operator APIs, the Cardano L1, or Sundial indexers) so this is primarily a convenience.

## Actions
### Generic L2 Transaction Endpoints
These closely follow the Midgard transaction model. As such most of these can be handled via the Cardano L1 or Layer Operator APIs.

- `POST /request`: Submit a new transaction request to the Sundial network.
- `POST /order`: Place a tx order.
- `POST /withdraw`: Initiate a withdrawal from the Sundial network to the Cardano L1.
- `POST /deposit`: Initiate a deposit from the Cardano L1 to the Sundial network.
- `POST /escape-hatch`: Initiate an escape hatch procedure to withdraw funds in case of an emergency. (Should never be used in normal operation.)
- `POST /:operatorId/request`: Submit a transaction request directly to a specific layer operator.

### Bridge Endpoints
These endpoints facilitate cross-chain transfers between Bitcoin and Cardano. They interact with the Bridge Operators to initiate and monitor bridge transfers.

- `POST /bridges/transferIn`: Initiate a bridge transfer between Bitcoin and Cardano.
- `POST /bridges/transferOut`: Initiate a bridge transfer from Cardano to Bitcoin.
- `POST /bridges/:bridgeId/transferIn`: Initiate a bridge transfer between Bitcoin and Cardano using a specific bridge.
- `POST /bridges/:bridgeId/transferOut`: Initiate a bridge transfer from Cardano to Bitcoin using a specific bridge.

### Yield Platform Endpoints
These endpoints allow users to interact with verified staking and yield-generating contracts on the Sundial network.
- `POST /stake/lock`: Lock tokens into a staking contract to earn yield.
- `POST /stake/unlock`: Unlock tokens from a staking contract.
- `POST /stake/update`: Update staking delegation.

## Informational Endpoints
These endpoints provide information about the Sundial network, operators, bridges, and user accounts, allowing for construction of queries with confidence.
### Addresses
- `GET /address/:address`: Retrieve the balance of a specific address on the Sundial network.

### Transactions
- `GET /transaction/:txid`: Retrieve the status of a specific transaction by its ID.
- `GET /address/:address/transactions`: List recent transactions associated with a specific address.

### Layer Operators
- `GET /schedule`: Retrieve the current schedule of layer operators.
- `GET /layer-operators`: List all active layer operators in the Sundial network.
- `GET /layer-operators/:operatorId`: Retrieve details about a specific operator.
- `GET /layer-operators/:operatorId/metrics`: Retrieve performance metrics for a specific operator.

### Bridges
- `GET /bridges`: List all active bridges.
- `GET /bridges/:bridgeId/metrics`: Retrieve performance metrics for a specific bridge.
- `GET /bridges/:bridgeId/bridge-operators`: List all active bridge operators.
- `GET /bridges/:bridgeId/bridge-operators/:operatorId`: Retrieve details about a specific bridge operator.
- `GET /bridges/:bridgeId/bridge-operators/:operatorId/metrics`: Retrieve performance metrics for a specific bridge operator.