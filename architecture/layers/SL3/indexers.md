# Sundial Indexers

A list of specifications around the needs of indexers on Sundial. These are intended to facilitate smooth communication and ensure implementations are complete.

Applications which follow these standards well may earn verification from the Sundial core team as _Verified Sundial Indexers (VSIs)_.

## General Requirements

1. **Compatibility**: Indexers should stay compatible with the latest version of the Sundial protocol.
2. **Performance**: Indexers should be optimized for performance, with low latency and high throughput.
3. **Security**: Indexers must implement best security practices to protect user data and prevent unauthorized access.

## Query Specifications

In the interest of facilitating smooth communication between indexers and wallets/clients, the following query specifications are defined:

### RPC

The specification is an extended version of the [UTxORPC](https://utxorpc.org/) standard.

The full, evolving specification is documented in the [Indexer UTxORPC](https://github.com/sundial-protocol/Indexer-UTxORPC) repository.

### REST

The OpenAPI Specification defines a standard interface for REST APIs, allowing for easy integration and interaction with indexers. It's included here in [sundial-indexer-openapi](./sundial-indexer-openapi.yaml).

The full, evolving specification is documented in the [Indexer OpenAPI](https://github.com/sundial-protocol/Indexer-OpenAPI) repository.
