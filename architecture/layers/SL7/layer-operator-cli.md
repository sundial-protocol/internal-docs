# Sundial Layer CLI Specification

## Setup
### CLI Commands

The Command-Line Interface (CLI) tools for managing operator services within the Sundial protocol. These provide a range of commands for deploying, configuring, monitoring, and maintaining operator nodes.

Planned commands include:

```
  --topology FILEPATH      The path to a file describing the topology.
  --database-path FILEPATH Directory where the state is stored.
  --immutable-database-path FILEPATH
                           Directory where the state is stored.
  --volatile-database-path FILEPATH
                           Directory where the state is stored.
  --validate-db            Validate all on-disk database files
  --socket-path FILEPATH   Path to a sundial-layer-node socket
  --watcher-socket-path-accept FILEPATH
                           Accept incoming watcher connection at local
                           socket
  --watcher-socket-path-connect FILEPATH
                           Connect to watcher listening on a local socket
  --bridge-socket-path-accept FILEPATH
                           Accept incoming sundial-bridge-node connection at local
                           socket
  --bridge-socket-path-connect FILEPATH
                           Connect to sundial-bridge-node listening on a local socket
  --config NODE-CONFIGURATION
                           Configuration file for the sundial-layer-node
  --non-producing-node     Start the node as a non block producing node even if
                           credentials are specified.
  --host-addr IPV4         An optional IPv4 address
  --host-ipv6-addr IPV6    An optional IPv6 address
  --port PORT              The port number
  --logs                   Path to the log files
  --status                 Show status & performance metrics
  -h,--help                Show this help text
```

### Configuration

When configuring a Layer node, a JSON file is expected which satisfies the following pattern:

```Typescript
export const LayerNodeConfig = S.Struct({
  // Node configuration
  node: S.Struct({
    endpoint: S.String.pipe(S.pattern(/^https?:\/\/.+/)),
  }),

  // Transaction generator configuration
  generator: S.Struct({
    enabled: S.Boolean,
    maxConcurrent: S.Number.pipe(S.positive(), S.int()),
    batchSize: S.Number.pipe(S.positive(), S.int()),
    intervalMs: S.Number.pipe(S.positive(), S.int()),
  }),

  // Logging configuration
  logging: S.Struct({
    level: S.Literal('debug', 'info', 'warn', 'error'),
    format: S.Literal('json', 'pretty'),
  }),
});
```

## Setting up environment variables

### SUNDIAL_NODE_SOCKET_PATH

Sundial CLI uses the *node-to-client* protocol to communicate with the node. This requires setting an environment variable for the node socket path. Ensure you use the path declared when starting the node.

```bash
export SUNDIAL_NODE_SOCKET_PATH=~/node.socket
```

### SUNDIAL_NODE_NETWORK_ID

Each network has a unique identifier (--mainnet or --testnet-magic NATURAL). This is used by the node-to-client protocol to ensure communication with a node on the desired network. It is useful to set up an environment variable for the network ID. Alternatively, you can provide the flag `--testnet-magic <network-id>` with each command that interacts with the node.  

- **Mainnet**

```bash
export SUNDIAL_NODE_NETWORK_ID=mainnet 
```

- **Testnet**

```bash
export SUNDIAL_NODE_NETWORK_ID=1
```

## Generating keys and addresses

### Generate a payment key pair and an address

To generate a key pair, run:

```shell
sundial-cli address key-gen \
--verification-key-file payment.vkey \
--signing-key-file payment.skey
```

### Build an address

This address will not have staking rights. It cannot delegate or receive rewards because it does not have a stake part associated with it, only a payment part (see [CIP-19](https://cips.cardano.org/cips/cip19/)).

```shell
sundial-cli address build \
--payment-verification-key-file payment.vkey \
--out-file paymentNoStake.addr
```

```shell
cat paymentNoStake.addr
addr_test1vzdtyyt48yrn2fa3wvh939rat0gyv6ly0ljt449sw8tppzq84xstz
```

### Generate a stake key pair

```shell
sundial-cli latest stake-address key-gen \
--verification-key-file stake.vkey \
--signing-key-file stake.skey
```

### Build the address with payment and stake parts

The resulting address will be associated with the **payment** and **stake** credentials:

```shell
sundial-cli address build \
--payment-verification-key-file payment.vkey \
--stake-verification-key-file stake.vkey \
--out-file payment.addr
```

```shell
cat payment.addr
addr_test1qzdtyyt48yrn2fa3wvh939rat0gyv6ly0ljt449sw8tppzrcc3g0zu63cp6rnjumfcadft63x3w8ds4u28z6zlvra4fqy2sm8n
```

### Query the balance of an address

```shell
sundial-cli query utxo --address $(< paymentNoStake.addr)
                           TxHash                                 TxIx        Amount
--------------------------------------------------------------------------------------
262c7891f932cde390bcc04c25805f3f422c1a5687d5d47f6681e68bb384fe6d     0        10000000000 lovelace + TxOutDatumNone
```