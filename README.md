# internal-docs

A repo to house private documentation and anything that doesn't fit cleanly into the docs repo

These documents are currently split into the following categories:

- Architecture
  - Layers
  - Roles
- Reports
- Developer Docs

`/architecture/` documents are intended to provide a high-level overview of the protocol, its components, and how they interact.

`/architecture/layers/` contains documents that describe the different layers of the protocol, their functions, and how they interact with each other.

`/architecture/roles/` contains documents that describe the different roles within the Sundial ecosystem, their responsibilities, and how they contribute to the overall functioning of the protocol.

`/reports/` contains various reports related to the project, including Catalyst reviews and feedback.

**Developer Docs** for [`sundial-node`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-node) and [`sundial-sdk`](https://github.com/sundial-protocol/sundial-monorepo/tree/main/demo/midgard-sdk) live at the
top level of this repo: [api.md](./api.md) (the node's HTTP API),
[smart-contracts.md](./smart-contracts.md) (how the node/SDK relate to the
on-chain Midgard validators), [security.md](./security.md) (integrator-facing
security best practices), and [environment.md](./environment.md)
(configuration reference), alongside the broader [architecture.md](./architecture.md).

Catalyst Reviewers, please see the [Milestone 4 Hyperlink Guide](./reports/milestone-guides/M4%20Hyperlink%20Guide.pdf) for a handy overview of where to find everything you're looking for.
