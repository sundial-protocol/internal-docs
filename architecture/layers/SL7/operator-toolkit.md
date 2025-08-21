# Operator Toolkit

The Operator Toolkit is a comprehensive suite of tools and resources designed to help users set up and manage their own operator services within the Sundial protocol.

There are 2 types of operator nodes:

- **Sundial Layer Node**: Responsible for processing and validating transactions within the Sundial network.
- **Sundial Bridge Node**: Facilitates transfer of assets between the Sundial Layer 2 and external UTXO networks.

For each type of operator node, the toolkit includes AT LEAST:

- **Documentation**: Detailed guides and references to help operators understand the protocol and its components.
- **Command-Line Interface (CLI) Tools**: Powerful tooling for managing operator services, including deployment, monitoring, and maintenance tasks.
- **Containerized Applications**: Pre-packaged applications that can be easily deployed in various environments, simplifying the setup process for new operators.

## Sundial-Layer-Node

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

## Sundial-Bridge-Node

The Bridge Node is partially blocked as we wait for details on the bridging solutions we're working with. As such the information here is highly prone to change.

### CLI Commands

The CLI for the bridge node should handle everything needed to

- Manage bridge connections and data flow
- Check bridge performance and health
- Monitor bridge logs and events

```
  --topology FILEPATH      The path to a file describing the topology.
  --database-path FILEPATH Directory where the state is stored.
  --immutable-database-path FILEPATH
                           Directory where the state is stored.
  --volatile-database-path FILEPATH
                           Directory where the state is stored.
  --validate-db            Validate all on-disk database files
  --socket-path FILEPATH   Path to a sundial-bridge-node socket
  --watcher-socket-path-accept FILEPATH
                           Accept incoming watcher connection at local
                           socket
  --watcher-socket-path-connect FILEPATH
                           Connect to watcher listening on a local socket
  --config NODE-CONFIGURATION
                           Configuration file for the sundial-bridge-node
  --host-addr IPV4         An optional IPv4 address
  --host-ipv6-addr IPV6    An optional IPv6 address
  --port PORT              The port number
  --logs                   Path to the log files
  --status                 Show status & performance metrics
  -h,--help                Show this help text
```

### Configuration

When configuring a Bridge node, a JSON file is expected which satisfies the following pattern:

```Typescript
export const BridgeNodeConfig = S.Struct({
  // Node configuration
  node: S.Struct({
    endpoint: S.String.pipe(S.pattern(/^https?:\/\/.+/)),
  }),

  // Logging configuration
  logging: S.Struct({
    level: S.Literal('debug', 'info', 'warn', 'error'),
    format: S.Literal('json', 'pretty'),
  }),
});
```

## Containerized Applications

We provide a unified approach with containerized applications that can be easily deployed in various environments, simplifying the setup process for new operators.

### Nix

```bash
git clone https://github.com/IntersectMBO/sundial-layer-node
cd sundial-layer-node
git switch -d tags/<TAGGED VERSION>
nix build .#sundial-layer-node
```

or

```bash
git clone https://github.com/IntersectMBO/sundial-bridge-node
cd sundial-bridge-node
git switch -d tags/<TAGGED VERSION>
nix build .#sundial-bridge-node
```

### Docker

```bash
docker run -d --name sundial-layer-node -p 8080:8080 sundial-protocol/sundial-layer-node/<TAGGED VERSION>
```

or

```bash
docker run -d --name sundial-bridge-node -p 8080:8080 sundial-protocol/sundial-bridge-node/<TAGGED VERSION>
```

## Documentation

Documentation should be comprehensive and user-friendly, providing clear instructions and examples for operators to effectively utilize the toolkit. It should cover installation, configuration, and usage of the various components, including the Layer Node and Bridge Node.

It should also be easy to download for offline use, and easy to reference.

While the emphasis should be on breadth & depth of content, operator documentation should still be structured to be approachable for users of varying experience levels.
