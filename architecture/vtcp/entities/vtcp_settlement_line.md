# Settlement Line
Implements an accounting primitive for the vTCP Network that manages the financial relationship between two interconnected [vTCP Nodes](/architecture/common/entities/vtcp_network_node.md). Comprises several key components:

- **Max Positive Balance:** The predefined maximum transfer limit.
- **Max Negative Balance:** The predefined maximum transfer limit.
- **Line Balance:** Tracks the net financial position between the Node and Contractor.
- **Transaction Log:** Maintains a comprehensive record of all financial transactions processed through the line.

The flow of funds is derived from these components:

- **Max positive flow:** Represents the maximum amount the Contractor can pay to the Node, calculated as:
  _Max positive flow = Max Positive Balance - Current Balance_

- **Max negative flow:** Represents the maximum amount the Node can pay to the Contractor, calculated as:
  _Max negative flow = Max Negative Balance + Current Balance_

## Asset Support

Each settlement line supports a single asset type. Multi-asset settlements between two nodes require separate settlement lines for each asset.

## Rebalancing Mechanism

Settlement line balances are updated through cryptographic co-signing between both parties. When both nodes agree on a new balance and sign it cryptographically, the rebalancing is considered complete. This bilateral agreement mechanism operates entirely at the L2 level without requiring on-chain operations.

The protocol also supports [circular debt clearing](/architecture/vtcp/protocols/protocol_circular_debt_clearing.md) in scenarios where multiple settlement lines form a cycle (e.g., A→B→C→A), though this is distinct from bilateral rebalancing.

## Capacity Modification

Settlement line capacity limits (Max Positive Balance and Max Negative Balance) can be modified dynamically through bilateral agreement between both nodes. Each node independently decides its own capacity limits.

**Minimum Capacity:** There is no minimum capacity requirement. Capacity limits can be set to zero for either or both directions.

**User-Hub Channels with Federation Backing:** When a user establishes a settlement line with a [Hub](/architecture/common/entities/hub.md) that is backed by [Federation](/architecture/common/entities/federation_btc.md) custody, capacity can be modified through L1 operations:
- **Capacity increase:** Via [deposit protocol](/architecture/federation/protocols/btc-vtcp-custody/protocol_btc<>vtcp_custody_deposit.md) - user deposits BTC on L1, Federation holds custody, Hub issues tokens on L2
- **Capacity decrease:** Via [redemption protocol](/architecture/federation/protocols/btc-vtcp-custody/protocol_btc<>vtcp_custody_redemption.md) - user withdraws, Hub returns tokens on L2, Federation releases BTC on L1