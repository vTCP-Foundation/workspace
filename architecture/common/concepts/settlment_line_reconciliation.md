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

## Canonical Hash: `reconciliation_hash`

To provide a single, deterministic, and globally unique identifier for any given reconciliation state, a canonical hash (`reconciliation_hash`) is computed. This hash serves as the definitive fingerprint of the agreement and is used in protocols that require a stable identifier for a specific state, such as the [Cooperative Redemption Flow](/architecture/federation/protocols/BTC%20<->%20vTCP%20Custody%20Protocol%20-%20redemption.md).

The hash is computed by both parties as follows:

`reconciliation_hash = BLAKE2b-256(lexicographically_smaller_signature || lexicographically_larger_signature)`

- **Lexicographical Signature Ordering:** To ensure the hash is identical regardless of which party computes it, the two signatures (`user_signature` and `hub_signature`) are first compared byte-wise. The signature with the smaller byte value is placed first in the concatenation, followed by the larger one. This removes any ambiguity related to the perspective of the participants.

This construction is sufficient because the signatures themselves are a cryptographic commitment to the full reconciliation payload (including `channel_id` and `sequence_number`), making their inclusion in the hash redundant. The hash of the ordered signatures provides the most minimal and secure fingerprint of the unique, co-signed state.

## Functional Role

A Settlement Channel Reconciliation serves several key functions within the network's operational protocols:

1.  **Auditing and Archiving:** It enables the periodic compression of a long transaction history into a single, cryptographically signed document. This practice minimizes data storage and verification overhead while upholding the integrity of financial obligations between parties.
2.  **State Synchronization:** It acts as a crucial synchronization point for all parties involved in a channel. This is essential for coordinating state-sensitive operations, such as fund withdrawals or channel rebalancing.
3.  **Dispute Resolution:** In the event of a disagreement or a non-cooperative scenario, the reconciliation with the highest sequence number and valid signatures serves as the definitive, irrefutable proof of the channel's most recent state.

## Implementation Details

```protobuf
message DebtReceipt {
    // The version of the protocol, e.g., 1, by default.
    uint16 protocol_version = 1;

    uint256 amount = 2;
    bytes signature = 3;
}
```

```protobuf
message ChannelReconciliation {
    // The version of the protocol, e.g., 1, by default.
    uint16 protocol_version = 1;

    // The globally unique and immutable identifier for the channel assigned by the Federation.
    uint64 channel_id = 2;
    
    // The sequence number of this state. Must be monotonically increasing.
    uint64 sequence_number = 3;

    // Max positive balance in the channel (in satoshis for vBTC).
    uint256 max_positive_balance = 4;
    
    // Max negative balance in the channel (in satoshis for vBTC).
    uint256 max_negative_balance = 5;
    
    // Balance in the channel (in satoshis for vBTC).
    uint256 balance = 6;

    bytes own_signature = 7;
    bytes contractor_signature = 8;
    
    // Optional key-value string fields for extensible metadata.
    // This is used to attach arbitrary metadata to the reconciliation, like 
    // - `channel_id` assigned by the BTC Federation, or
    // - `memo` field for the deposit flow.
    map<string, string> metadata = 10;
}
```