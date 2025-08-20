# Operator Toolkit

The Operator Toolkit is a comprehensive suite of tools and resources designed to help users set up and manage their own operator services within the Sundial protocol. It includes:

- **Documentation**: Detailed guides and references to help operators understand the protocol and its components.
- **Command-Line Interface (CLI)**: A powerful CLI for managing operator services, including deployment, monitoring, and maintenance tasks.
- **Graphical User Interface (GUI)**: A user-friendly GUI for interacting with the operator services, making it easier to manage and monitor operations.
- **Containerized Applications**: Pre-packaged applications that can be easily deployed in various environments, simplifying the setup process for new operators.

The Operator Toolkit aims to streamline the onboarding process for new operators, providing them with the necessary tools and resources to succeed in the Sundial ecosystem.

## CLI
The Command-Line Interface (CLI) is a powerful tool for managing operator services within the Sundial protocol. It provides a range of commands for deploying, configuring, monitoring, and maintaining operator nodes.

### Commands
TODO: List of CLI commands and their descriptions.

## GUI
TODO: Description of the Graphical User Interface (GUI) for managing operator services.

## Containerized Applications
Running an operator should be as simple as running a single command in your terminal or clicking a button in a GUI. We provide containerized applications that can be easily deployed in various environments, simplifying the setup process for new operators.

### Nix
```bash
nix build github:sundial-protocol/sundial-l2-node/<TAGGED VERSION>
```
or
```bash
nix build github:sundial-protocol/sundial-bridge-node/<TAGGED VERSION>
```

### Docker
```bash
docker run -d --name sundial-l2-node -p 8080:8080 sundial-protocol/sundial-l2-node/<TAGGED VERSION>
```
or
```bash
docker run -d --name sundial-bridge-node -p 8080:8080 sundial-protocol/sundial-bridge-node/<TAGGED VERSION>
```