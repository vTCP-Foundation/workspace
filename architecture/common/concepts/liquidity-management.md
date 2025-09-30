# Liquidity Management

## Overview

vTCP's liquidity management system enables flexible, off-chain balance updates across the network without requiring on-chain transactions. This approach significantly reduces costs and latency compared to traditional blockchain-based payment channels.

## Settlement Line Rebalancing

### Bilateral Agreement Mechanism

vTCP [settlement lines](../../workspace/architecture/common/entities/vtcp_settlement_line.md) operate as bidirectional accounting primitives between node pairs. When both parties agree on a new balance and cryptographically sign it, the rebalancing is immediately effective at the L2 level.

**Key characteristics:**
- **No on-chain operations required** - All rebalancing occurs through cryptographic co-signing
- **Instant settlement** - Balance updates are effective immediately upon bilateral agreement
- **Zero transaction fees** - No blockchain fees for rebalancing operations
- **Full flexibility** - Any balance adjustment can be proposed and agreed upon

### Circular Debt Clearing

Beyond bilateral rebalancing, vTCP supports [circular debt clearing](../../workspace/architecture/vtcp/protocols/protocol_circular_debt_clearing.md) for scenarios where multiple settlement lines form cycles (e.g., A→B→C→A).

This mechanism identifies circular relationships and optimizes net obligations across all participating nodes simultaneously, further improving capital efficiency.

## Capacity Management

### Dynamic Capacity Adjustment

Settlement line capacity limits can be modified dynamically through bilateral agreement:

- **Independent capacity limits** - Each node sets its own Max Positive Balance and Max Negative Balance
- **Bilateral modification** - Both parties must agree to capacity changes
- **No minimum requirement** - Capacity can be set to zero in either or both directions
- **Flexible scaling** - Capacity can increase or decrease based on trust and liquidity needs

### User-Hub Channels with Federation Backing

When users establish settlement lines with [Hubs](../../workspace/architecture/common/entities/hub.md) that are backed by [BTC Federation](../../workspace/architecture/common/entities/federation_btc.md) custody, channel capacity can be modified through L1 operations via the [BTC ⟷ vTCP Custody Protocol](../../workspace/architecture/federation/protocols/btc-vtcp-custody/protocol_btc<>vtcp_custody_deposit.md):

**Deposit Flow (Capacity Increase):**
1. User deposits BTC on L1 to Federation custody
2. Federation confirms deposit and notifies Hub
3. Hub expands settlement line capacity with user on L2
4. Hub issues corresponding vBTC tokens to user

**Redemption Flow (Capacity Decrease):**
1. User initiates withdrawal via [redemption protocol](../../workspace/architecture/federation/protocols/btc-vtcp-custody/protocol_btc<>vtcp_custody_redemption.md)
2. Hub returns tokens on L2, reducing settlement line capacity
3. Federation releases corresponding BTC to user on L1

This architecture creates a direct bridge between L1 liquidity and L2 channel capacity, allowing users to seamlessly move funds between layers while the Hub maintains backing through Federation custody.

## Multi-Asset Support

### One Asset Per Settlement Line

Each settlement line supports a single asset type. For multi-asset relationships between two nodes:

- **Separate lines required** - Nodes establish distinct settlement lines for each asset
- **Independent balances** - Each asset maintains its own balance and capacity limits
- **Parallel operation** - Multiple settlement lines between the same node pair operate independently

### Cross-Asset Payments

While individual settlement lines are single-asset, the vTCP network supports cross-asset payments through multi-hop routing:

- **Automatic conversion** - Payments can traverse nodes that exchange between assets
- **Optimal path finding** - [Route discovery](../../workspace/architecture/vtcp/protocols/protocol_route_discovery.md) algorithms identify the most efficient paths considering both routing fees and conversion rates
- **Transparent to sender** - Users can send in one asset while recipients receive in another

**Example:** A user holding USDC can pay a merchant accepting USDT by routing through nodes that provide USDC/USDT exchange services along the path.

## Comparison with Lightning Network

### Liquidity Lockup

**Lightning Network:**
- Channel capacity is fixed at opening
- Funds are locked in specific channels
- Rebalancing requires complex circular payments or submarine swaps
- Increasing capacity requires closing and reopening channels (on-chain cost)

**vTCP:**
- Settlement line balances update through cryptographic agreement
- Capacity can be modified bilaterally without on-chain operations
- Hub-Federation channels support seamless capacity expansion via custody protocol
- Circular debt clearing optimizes multiple channels simultaneously

### Capital Efficiency

vTCP's flexible rebalancing and capacity management enables significantly higher capital efficiency than fixed-capacity Lightning channels, particularly for Hub operators serving many users.

## Technical References

- [Settlement Line Entity](../../workspace/architecture/common/entities/vtcp_settlement_line.md)
- [Channel Rebalancing Protocol](../../workspace/architecture/vtcp/protocols/protocol_channel_rebalancing.md)
- [Circular Debt Clearing Protocol](../../workspace/architecture/vtcp/protocols/protocol_circular_debt_clearing.md)
- [Route Discovery Protocol](../../workspace/architecture/vtcp/protocols/protocol_route_discovery.md)
- [BTC ⟷ vTCP Custody Protocol](../../workspace/architecture/federation/protocols/btc-vtcp-custody/protocol_btc<>vtcp_custody_deposit.md)