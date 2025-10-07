# Component Interactions / Data Flow

This document contains sequence diagrams illustrating the flow of assets and data between the various components of the Sundial architecture. For more detailed information on each component, please refer to the respective architecture documents.

## Integration Points

```mermaid
sequenceDiagram
    participant User
    participant UI as Sundial Web UI
    participant L1 as Any UTxO L1
    participant Bridge as Bridging Flow
    participant L2 as Sundial L2
    participant Stake as Staking service
    participant CL1 as Cardano L1
    participant Yield as Yield Instruments
    
    Note over User, Bridge: Any UTxO L1
    Note over Bridge, Yield: Sundial L2

    User->>UI: Initiate Action
    UI->>L1: Interact with L1
    L1->>Bridge: Bridge Assets 
    Bridge->>L2: Receive Assets
    L2->>Stake: Stake Assets
    L2->>CL1: Settlement
    Stake->>Yield: Invest in Yield Instruments
    Yield->>Stake: Yield Returns
    Stake->>L2: Rewards/Returns
    L2->>User: Finalize Action

    User->>UI: Monitor/Manage
    UI->>L2: Query State/Balance
    L2->>User: Provide Updates

    User->>UI: Initiate Withdrawal
    UI->>L2: Create Withdrawal Tx
    L2->>Bridge: Prepare for Bridging
    Bridge->>L1: Release Assets
    L1->>User: Complete Withdrawal

```

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

    Note over User, BridgeOp: Bitcoin L1
    Note over BridgeOp, L1: Sundial L2

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


    Note over User, BridgeOp: Sundial L2
    Note over BridgeOp, BTCL1: Bitcoin L1

    User->>UI: Initiate Bridge Request
    UI->>BridgeService: Select Bridge

    BridgeService->>L2: Burn Request
    BridgeOp->>L2: Burn
    BridgeOp->>L2: Mint Wrapped BTC
    L2->>L1: Settle Tx
    L1->>BridgeOp: Mithril Signatures
    BridgeOp->>BTCL1: Release BTC


```

## Deposit Flow (Cardano to Sundial)

```mermaid
sequenceDiagram
    participant User as User on L1
    participant UI
    participant L1 as Settlement Queue
    participant BP as Block Producer
    participant Facilitator
    participant L2 as User on L2

    Note over User, Facilitator: Cardano L1
    Note over BP, L2: Sundial L2
    User->>UI: Initiate Deposit Request
    UI->>L1: Create Deposit Tx
    BP->>L1: Monitor for Deposits

alt Standard
    BP->>L2: Funds to User
else Accelerated
    Facilitator->>L1: Monitor for Deposits
    Facilitator->>L2: Accelerated Funds
    BP->>Facilitator: Settle Tx on L2
end
```

## Withdrawal Flow (Sundial to Cardano)
```mermaid
sequenceDiagram
    participant L2 as User on L2
    participant UI
    participant BP as Block Producer
    participant Facilitator
    participant L1 as State Queue
    participant User as User on L1

    Note over L2, Facilitator: Sundial L2
    Note over BP, User: Cardano L1
    L2->>UI: Initiate Withdrawal Request
    UI->>BP: Create Withdrawal Tx
    BP->>L1: Tx Inclusion
alt Standard
    L1->>User: Funds Available after Challenge Period
else Accelerated
    Facilitator->>BP: Monitor for Withdrawals
    Facilitator->>User: Accelerated Funds
    L1->>Facilitator: Settle Tx on L1
end
```