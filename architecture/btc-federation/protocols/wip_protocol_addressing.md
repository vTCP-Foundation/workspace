# vTCP Network Addressing Scheme

_Mykola Ilaschuk, Dima Chizhevsky, 🤖 Gemini 2.5 Pro, 🤖 Claude 4 Sonnet_
<br>
_v0.1, 2025-07-01_
_**STATE: WIP**_
<br>
<br>

## 1. Overview

This document defines the comprehensive addressing and identity framework for all entities within the [vTCP Network (L2)](/architecture/common/entities/vtcp_network.md). It specifies how entities are uniquely identified, located for peer-to-peer communication, and how they manage cryptographic keys for interactions with external L1 blockchains, such as the [Bitcoin Network (L1)](/architecture/common/entities/btc_network.md).

The vTCP protocol's internal operations, particularly within state channels, primarily rely on one-time Lamport Signatures. This choice avoids the complexities of public-key infrastructure for state updates. However, to achieve interoperability with external networks, a robust system for managing and discovering conventional public keys is essential. This document describes that system.

## 2. Core Concepts

### 2.1. vTCP Entity Identifier (`vID`)

The `vID` is the canonical, unique identifier for any entity on the vTCP network, including [Nodes](/architecture/common/entities/vtcp_network_node.md), [Hubs](/architecture/common/entities/vtcp_network_hub.md), and the [Federation](/architecture/common/entities/btc_federation.md) itself.

- **Uniqueness:** Each `vID` is guaranteed to be unique across the entire network.
- **Persistence:** An entity's `vID` is persistent and does not change over its lifetime.
- **Function:** It serves as the primary key for looking up an entity's data in the network registry and as a stable identifier for routing and communication.

### 2.2. Network Endpoint Address

While the `vID` identifies *who* an entity is, the Network Endpoint Address specifies *how* to reach it for direct L2 communication. This address is typically in an `ip:port` format.

- **Dynamic Nature:** An entity's network endpoint may change over time (e.g., due to network reconfiguration or migration).
- **Discovery:** Entities discover each other's current network endpoint by querying the Entity Registry using the target's `vID`.

### 2.3. Role of Lamport Signatures

The vTCP protocol uses Lamport Signatures for signing state updates within [Settlement Lines](/architecture/common/entities/vtcp_settlement_line.md).

- **One-Time Use:** Lamport keys are used once and then discarded, providing perfect forward secrecy for channel states.
- **Not for Identity:** Due to their one-time nature, they are unsuitable for creating a persistent, publicly-known identity. This distinction is critical: Lamport signatures secure channel history, while the `vID` and its associated keys secure identity and cross-chain interactions.

### 2.4. Registered External Public Keys

To facilitate cross-chain operations like deposits and redemptions as described in the [BTC <-> vTCP Custody Protocol](/architecture/btc-federation/protocols/BTC%20<->%20vTCP%20Custody%20Protocol.md), entities must be able to associate stable public keys from external networks with their `vID`.

- **Purpose:** These keys allow other entities (e.g., the Federation) to construct valid L1 transactions payable to the entity. For instance, for a BTC redemption, the Federation needs the user's `BTC` public key to create the payout transaction.
- **Supported Networks:** The system is designed to be extensible, allowing for the registration of keys for various blockchains (e.g., `BTC`, `ETH`).
- **Registration:** An entity registers its external public keys in the Entity Registry. The process for adding or updating a key must be authenticated to prevent unauthorized changes.

## 3. The Entity Registry

The [BTC Federation](/architecture/common/entities/btc_federation.md) is responsible for maintaining a central, authoritative Entity Registry. This registry serves as the network's directory service.

**Registry Schema (for each entity):**
- `vID`: (Primary Key) The unique identifier of the entity.
- `EntityType`: (e.g., `Node`, `Hub`).
- `NetworkEndpoint`: The current `ip:port` for L2 communication.
- `ExternalKeys`: A map of network names to public keys.
  - `BTC`: The user's Bitcoin public key (e.g., in `secp256k1` format).
  - `ETH`: The user's Ethereum public key.
  - ... and so on for other supported chains.

## 4. Proposed Addressing URI Scheme

To standardize how entities are referenced, we propose a URI scheme.

`vtcp://<vID>[@<host>:<port>]`

- **`vtcp://`**: The protocol scheme.
- **`<vID>`**: The mandatory unique identifier of the entity.
- **`[@<host>:<port>]`**: The optional, and potentially outdated, network location. Relying on the registry for the current location is the recommended practice.

## 5. Security Considerations

- **Registry Integrity:** The Entity Registry is a critical piece of infrastructure. All write operations (registration, updates to endpoints or keys) MUST be authenticated, for example, via a signature from a master key associated with the `vID` at the time of its creation.
- **Key Rotation and Revocation:** The protocol must define a secure process for an entity to update or revoke a registered external public key. This process is critical to recover from key compromise. It should require a higher level of authentication than standard operations.
- **Liveness and Endpoint Accuracy:** Stale endpoint information can lead to routing failures. Nodes should periodically refresh their endpoint information with the registry. The registry could also perform periodic liveness checks.
- **Privacy:** While the `vID` and associated public keys are necessarily public for the system to function, users should be encouraged to use fresh, single-purpose addresses/keys for L1 transactions to enhance privacy, and then register those keys for specific operations.
