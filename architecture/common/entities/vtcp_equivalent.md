# vTCP Equivalent

_Asset type representation and exchange framework in the vTCP Network_

## Definition

A **vTCP Equivalent** represents any type of asset, currency, or tradeable unit that can be accounted for, exchanged, and transferred within the vTCP Network. This includes digital assets, fiat currencies, and any other asset type that can potentially have a ticker symbol and be subject to accounting operations.

## Core Concept

In the vTCP system, "equivalent" refers to the mathematical and functional treatment of different asset types as interchangeable units of value. While a Bitcoin (BTC) and a US Dollar (USD) are fundamentally different assets, they are treated as "equivalent" from the protocol's perspective in that they:

- Can be held in [settlement lines](/architecture/common/entities/vtcp_settlement_line.md) between network participants
- Can be exchanged at market-determined rates
- Serve equivalent functions as units of account and value transfer
- Are subject to the same flow calculation algorithms
- Follow identical protocol rules for topology collection and path finding

## Supported Asset Types

vTCP Equivalents encompass a broad range of asset types:

### Digital Assets
- **Cryptocurrencies**: BTC, ETH, and other blockchain-native tokens
- **Stablecoins**: USDC, USDT, DAI, and other digital dollar representations
- **Central Bank Digital Currencies (CBDCs)**: Digital representations of national currencies

### Traditional Assets
- **Fiat Currencies**: USD, EUR, JPY, GBP, and other national currencies
- **Commodities**: Gold, silver, oil, and other tradeable commodities
- **Securities**: Stocks, bonds, and other financial instruments (where legally permissible)

### Custom Assets
- **Loyalty Points**: Airline miles, reward points, gaming tokens
- **Utility Tokens**: Access tokens, service credits, prepaid balances
- **Synthetic Assets**: Derivatives and other complex financial instruments

## Technical Implementation

### Serialized Representation
Each equivalent is represented by a `SerializedEquivalent` identifier that uniquely identifies the asset type across the network:

```cpp
SerializedEquivalent equivalentFrom;  // Source asset
SerializedEquivalent equivalentTo;    // Target asset
```

### Exchange Rate Framework
Equivalents can be exchanged through the exchange rate system:

- **Exchange Rates**: Define conversion ratios between different equivalents
- **Rate Precision**: Handled through value and shift mechanisms for accurate decimal representation
- **TTL Management**: Exchange rates have expiration timestamps for real-time accuracy
- **Capacity Limits**: Min/max exchange amounts define feasible conversion ranges

### Settlement Line Support
Each equivalent can be held in [settlement lines](/architecture/common/entities/vtcp_settlement_line.md) between network participants:

- **Multi-Equivalent Channels**: Single channels can contain settlement lines for multiple equivalents
- **Independent Balances**: Each equivalent maintains separate balance tracking
- **Flow Isolation**: Payment flows are calculated per equivalent with cross-equivalent bridging

## Payment Scenarios

### Same-Equivalent Payments
When payer and receiver use the same equivalent:
```
Payer: USD → Network → Receiver: USD
```
- Direct flow calculation without currency conversion
- Simplified topology collection
- No exchange rate requirements

### Cross-Equivalent Payments
When payer and receiver use different equivalents:
```
Payer: BTC → Exchange Node (BTC→USD) → Receiver: USD
```
- Multi-equivalent topology collection
- Exchange rate discovery and validation
- Optimal path calculation considering conversion costs

### Multi-Equivalent Support
Payers can utilize multiple equivalents for a single payment:
```
Payer: [BTC, ETH] → Exchange Nodes → Receiver: USD
```
- Parallel topology collection across payer equivalents
- Optimization across all available exchange paths
- Maximum flow calculation considering all options

## Design Principles

### Mathematical Equivalence
From the protocol's perspective, all assets are treated with mathematical equivalence:
- Same algorithms apply regardless of asset type
- Consistent precision and representation standards
- Uniform security and validation requirements

### Extensibility
The equivalent system is designed for future expansion:
- New asset types can be added without protocol changes
- Flexible serialization supports various identifier schemes
- Modular exchange rate mechanisms accommodate different pricing sources

### Interoperability
Equivalents enable cross-network and cross-asset operations:
- Bridge different blockchain networks
- Connect traditional finance with digital assets
- Enable complex multi-asset payment flows

## Future Enhancements

The equivalent framework provides foundation for advanced features:

- **Dynamic Equivalent Discovery**: Automatic detection of new asset types
- **Decentralized Exchange Integration**: Direct DEX connectivity for rate discovery
- **Regulatory Compliance**: Asset-specific compliance and reporting mechanisms
- **Advanced Derivatives**: Complex financial instruments built on equivalent infrastructure
- **Cross-Chain Bridging**: Native support for assets from multiple blockchain networks

## Related Documentation

- [vTCP Network](/architecture/common/entities/vtcp_network.md) - Network architecture overview
- [Exchange Topology Collection Protocol](/architecture/vtcpd/protocols/exchange-topology-collection-protocol.md) - Multi-equivalent topology collection
- [BTC Federation](/architecture/common/entities/btc_federation.md) - Bitcoin-backed equivalent implementation
- [Exchange Flow Calculation](/workflow/prd/vtcpd/04-exchange-flow-calculation.md) - Cross-equivalent payment optimization