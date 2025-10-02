# Component Interactions / Data Flow

## Bridge Flow (Bitcoin to Sundial)
Bridging can look pretty different based on the bridge used. This diagram illustrates the flow for both the Cardinal and Charms bridges.

```mermaid
sequenceDiagram
    participant User
    participant UI as Sundial Web UI
    participant BridgeService as Bridging Service
    participant BTCL1 as Bitcoin L1
    participant BridgeOp as Bridge Operator
    participant L2 as Sundial L2
    participant L1 as Cardano L1

    Note over User, L1: Bitcoin to Sundial Bridge Flow

    User->>UI: Initiate Bridge Request
    UI->>BridgeService: Select Bridge

    alt Cardinal
        BridgeService->>BTCL1: Lock Request
        BridgeService->>BridgeOp: Notify of Lock
        BridgeOp->>L2: Mint Wrapped BTC
        BridgeOp->>BTCL1: Lock Tx
    else Charms
        BridgeService->>L2: Create placeholder utxo
        BridgeService->>BTCL1: Create Beaming Transaction
        BTCL1->>L2: Consume Beaming Transaction
    end

    L2->>L1: Settle Tx
```

## Bridge Flow (Sundial to Bitcoin)
Unlocking for Charms & Cardinal bridges both rely on similar mechanisms involving Mithril threshold signatures.

```mermaid
sequenceDiagram
    participant User
    participant UI as Sundial Web UI
    participant BridgeService as Bridging Service
    participant L2 as Sundial L2
    participant L1 as Cardano L1
    participant BridgeOp as Bridge Operator
    participant BTCL1 as Bitcoin L1


    Note over User, BTCL1: Sundial to Bitcoin Bridge Flow

    User->>UI: Initiate Bridge Request
    UI->>BridgeService: Select Bridge

    BridgeService->>L2: Burn Request
    BridgeOp->>L2: Burn
    BridgeOp->>L2: Mint Wrapped BTC
    L2->>L1: Settle Tx
    L1->>BridgeOp: Mithril Signatures
    BridgeOp->>BTCL1: Release BTC


```

## Settlement Flow
TODO

