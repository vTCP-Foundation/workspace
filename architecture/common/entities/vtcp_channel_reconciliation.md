# Settlement Channel Reconciliation

**Channel Reconciliation** is a cryptographically secured snapshot of a [Settlement Channel's](/architecture/common/entities/vtcp_settlment_channel.md) state, capturing the financial obligations between two vTCP network participants at a specific point in time. It serves as an audit and synchronization tool, allowing the history of transactions to be compressed into a single, agreed-upon state.

## Key Components

A [Channel Reconciliation](/architecture/common/entities/vtcp_channel_reconciliation.md) contains the following data:

- **Channel Identifier:** A unique identifier for the [Settlement Channel](/architecture/common/entities/vtcp_settlment_channel.md) to which the reconciliation belongs.
- **Current Channel State:**
    - **Balance:** The final balance reflecting the financial obligations of the parties.
    - **Capacity:** The incoming and outgoing capacity of the channel at the time of reconciliation.
- **Cryptographic Signatures:** Digital signatures from both channel participants, confirming their agreement on the recorded state. This ensures that neither party can unilaterally alter or dispute the reconciliation.
- **Transaction Set:** A list of transactions that have occurred since the previous reconciliation. This provides full traceability and state verifiability. Can be empty if no transactions have occurred since the previous reconciliation.
- **Sequence Number (Nonce):** A strictly and monotonically increasing integer (e.g., `uint64`). It starts at 0 for the initial channel setup and must be incremented by exactly 1 for each subsequent reconciliation. This is the most critical field for preventing replay attacks. During any dispute, the reconciliation with the highest valid sequence number is considered the only source of truth. The Federation MUST reject any reconciliation with a sequence number lower than or equal to the one it has on record for a given channel.

## Functional Role

A Settlement Channel Reconciliation serves several key functions within the network's operational protocols:

1.  **Auditing and Archiving:** It enables the periodic compression of a long transaction history into a single, cryptographically signed document. This practice minimizes data storage and verification overhead while upholding the integrity of financial obligations between parties.
2.  **State Synchronization:** It acts as a crucial synchronization point for all parties involved in a channel. This is essential for coordinating state-sensitive operations, such as fund withdrawals or channel rebalancing.
3.  **Dispute Resolution:** In the event of a disagreement or a non-cooperative scenario, the reconciliation with the highest sequence number and valid signatures serves as the definitive, irrefutable proof of the channel's most recent state. 