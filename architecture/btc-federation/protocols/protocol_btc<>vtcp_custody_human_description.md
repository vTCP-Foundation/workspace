# BTC ⟷ vTCP Custody Protocol

_Mykola Ilaschuk, Dima Chizhevsky, 🤖 Gemini 2.5 Pro, 🤖 Claude 4 Sonnet_
<br>
_v0.1, 2025-06-30_
<br>
<br>

## Overview
This protocol enables the issuance, exchange, and redemption of **BTC-backed** vTCP tokens between the [Bitcoin Network (L1)](/architecture/common/entities/btc_network.md) and [vTCP Network (L2)](/architecture/common/entities/vtcp_network.md).

## Core Requirements

These requirements define the absolute, non-negotiable functions and security guarantees of the protocol.

### Functional Requirements

- **1. Representative Token Issuance:** The system shall enable users to lock a native asset on a Base Chain to mint an equivalent (1:1) amount of a representative token on a Token Chain.
- **2. Representative Token Redemption:** The system shall enable users to burn representative tokens to redeem an equivalent (1:1) amount of the native asset on the Base Chain. This must be supported via:
    - A **Cooperative Redemption** process for efficient, mutually-agreed-upon withdrawals.
    - A **Non-Cooperative Redemption** process for users to unilaterally reclaim their assets if a counterparty is unresponsive or malicious.
- **3. User State Protection:** The system must maintain a secure, real-time backup of the latest agreed-upon channel states, ensuring the funds of an offline user can be correctly identified and recovered.

### Security Requirements

- **1. Collateral Security:** All native assets locked on the Base Chain must be held in a manner that makes them cryptographically immune to unauthorized seizure, requiring authorization from the protocol's defined consensus mechanism.
- **2. State Integrity and Authenticity:** The system must ensure that the most recent, validly co-signed channel state is always treated as authoritative, preventing the use of outdated states to defraud participants.
- **3. Attack Resistance:** The protocol must be immune to replay attacks and incorporate economic or reputation-based deterrents against resource-exhaustion ("griefing") attacks.
- **4. Explicit Minimized Trust Assumptions & Transparency:** The protocol's security shall operate on a trust-minimized model, explicitly defined by an M-of-N validator set. It assumes that a qualified majority (M) of the N validators are honest and will not collude. To enforce this, all critical protocol actions and their cryptographic proofs must be transparent and publicly auditable.

## Supervisory and Liquidity Functions

Beyond the core issuance and redemption flows, the protocol incorporates advanced functions for system-wide integrity and operational efficiency.

- **1. State Channel Rebalancing:** The protocol shall provide a mechanism for liquidity providers (Hubs) to rebalance funds across multiple state channels. This process optimizes liquidity allocation without requiring on-chain transactions on the Base Chain, and it must require the verifiable, cryptographic consent of all affected users.

- **2. Verifiable Collateral Audits:** The system shall perform regular, automated audits to cryptographically prove that the total supply of the representative token does not exceed the total amount of locked native asset collateral. The audit mechanism must be structured to allow any third party, including individual users, to independently verify the integrity of the system's liabilities.

## Actors
- [BTC Federation](/architecture/common/entities/btc_federation.md)
- The Hub - [vTCP Hub](/architecture/common/entities/vtcp_network_hub.md)
- The User - [vTCP Node](/architecture/common/entities/vtcp_network_node.md)

## System Prerequisites

### Hub Security Bond
Hubs are required to lock a significant amount of their own capital as a security bond. This stake serves as an economic disincentive against malicious behavior. In the event of proven fraud or operational failure leading to a financial shortfall (such as a Hub Stake Overflow), this bond is automatically forfeited ("slashed") to cover the losses.

**Collateralization Rule:** The Hub's bonded collateral **MUST** always be **at least equal** to the aggregate BTC value backing all outstanding vTCP tokens it has issued (a strict **1:1 ratio** between total user deposits and the Hub's stake). If the bonded balance ever drops below this threshold, the Hub is automatically suspended from opening new channels, rebalancing existing channels, or issuing additional vTCP until the deficiency is remedied.

### Federation State Backup
The Federation is mandated to act as a real-time backup service for channel states. Node are expected to periodically push the latest co-signed state for every channel to the Federation (as part of paid subscription), independent of any specific flow. This "heartbeat" mechanism ensures the Federation always has a recent, valid state on record to use for adjudication, providing critical protection for users, especially those who may be offline.

## Protocol Flows

### Flow 1: Issuance Process

#### Overview
This flow enables a user to lock BTC on L1 and receive an equivalent amount of vTCP tokens on L2. The process covers both the creation of new settlement lines and the topping-up of existing ones, and is designed to be secure, flexible, and resilient. For a detailed technical specification, see the [Issuance Protocol Implementation Details](./BTC%20<->%20vTCP%20Custody%20Protocol%20-%20deposit.md).

#### Actors
- **User**: Initiates the deposit. Must have a registered `vID` for L2 communication.
- **Hub**: Provides L2 connectivity and liquidity. Must have a registered `vID`.
- **Federation**: Manages the protocol, validates requests, and publishes state changes.

#### Process Steps

**Step 1: Deposit Request**
- The User submits a signed `DepositRequest` to the Federation. This request includes their stable `vID` (for L2 routing) and is signed with the key of the L1 wallet they intend to deposit from, proving ownership of the funds for this specific transaction.
- The Federation validates the cryptographic signature on the request and computes a unique `request_id` for tracking. It treats the `vID` as an opaque identifier, forwarding it to the Hub. The Hub is responsible for validating the `vID` and establishing a connection with the user.

**Step 2: Authorization and Discovery**
- The Federation publishes a `DepositAuthorization` to a public location (e.g., the Federation Chain). This message contains the `request_id`, the user's `vID`, and the unique L1 multisig address for the deposit.
- Both the User and the selected Hub watch for this authorization. The Hub uses the `user_vID` to find the user's network address and prepare for an L2 connection.

**Step 3: L2 Channel Establishment**
- The Hub connects to the User's node to establish an L2 Settlement Line.
- They co-sign an initial `ChannelReconciliation` with a zero balance. Crucially, the L1 deposit address is embedded in the `memo` field of this reconciliation, creating a permanent link between the L1 and L2 states.

**Step 4: L1 Deposit**
- With the L2 channel active, the User deposits the BTC amount to the provided L1 address.
- The Federation monitors the Bitcoin network for this transaction.

**Step 5: L2 Token Issuance**
- After enough block confirmations, the Federation publishes a final `DepositConfirmation`.
- The Hub sees this confirmation and issues the vTCP tokens to the User on the L2 channel by co-signing a new reconciliation with the updated balance.

**Step 6: Handling Failures**
- **L2 Channel Fails:** If the channel isn't established, the authorization expires. No funds move, no fault is assigned.
- **User Abandons:** If the channel is made but the user never deposits, the Hub can submit the initial zero-balance reconciliation to the Federation to prove it fulfilled its part, which may affect the user's reputation.
- **Hub Fails to Issue Tokens:** If the user deposits but the Hub fails to issue tokens, the user can initiate the **Deposit Reversal Flow**. They submit a request to the Federation, which opens a challenge window. If the Hub cannot provide proof of issuance within this window, the Federation returns the L1 funds directly to the user.
- **Incorrect Deposit Amount:** If the user deposits an amount different from what was authorized, the Federation will not confirm the deposit. The process halts, and the user must initiate a separate, non-disputable refund process to reclaim their funds (less transaction fees).

### Flow 2: Cooperative Redemption

#### Overview
This flow enables a User to cooperatively redeem their vTCP tokens for an equivalent amount of BTC on L1. The process is designed to be fast and efficient. While the Federation treats every redemption request with the same level of security by default (initiating a challenge window), a cooperative redemption uses an "accelerate" mechanism where mutual consent from the User and Hub allows this window to be bypassed, ensuring a rapid settlement.

#### Actors
- [BTC Federation](/architecture/common/entities/btc_federation.md) - Manages the custody protocol and L1 funds.
- The Hub - [vTCP Hub](/architecture/common/entities/vtcp_network_hub.md) - The User's L2 counterparty.
- The User - [vTCP Node](/architecture/common/entities/vtcp_network_node.md) - Initiates the redemption.

#### Process Steps

**Step 1: Redemption Agreement & State Reconciliation**
- The User signals their intent to redeem a specific amount of vTCP tokens to the Hub.
- The User and Hub collaborate to create and co-sign a new [channel reconciliation](/architecture/common/entities/vtcp_channel_reconciliation.md). This reconciliation reflects the new channel state *after* the redemption amount has been deducted from the User's balance.
- *(Security: This co-signed state is the definitive, non-repudiable proof of the agreement.)*

**Step 2: User Submits Redemption Request**
- The User submits the co-signed channel reconciliation to the Federation, along with their desired L1 BTC destination address.
- The Federation validates the request and, by default, opens a challenge window to allow the Hub to contest it.

**Step 3: Hub Accelerates the Redemption**
- The Hub, having already agreed to the redemption, immediately submits the **exact same co-signed reconciliation** to the Federation.
- *(Efficiency: The Federation interprets the submission of an identical state from both parties as explicit, real-time consent. This action "accelerates" the process.)*

**Step 4: Federation Verification and Settlement**
- Upon receiving the matching state from the Hub, the Federation bypasses the remainder of the challenge window.
- It performs final verification on the state (e.g., sequence number, signatures).
- Upon successful verification, the Federation proceeds with two atomic actions:
  1. **L1 Transaction:** It unlocks the specified amount of BTC from its custody and broadcasts the transaction to the User's L1 address.
  2. **L2 Update:** It updates its internal registry to reflect the new, post-redemption state of the [vTCP settlement line](/architecture/common/entities/vtcp_settlement_line.md).
- The redemption is considered complete once the L1 transaction is confirmed on the Bitcoin network.

### Flow 3: Non-Cooperative Redemption

#### Overview
This flow serves as a critical security mechanism, allowing a User to unilaterally force the closure of a settlement line if the Hub is unresponsive, malicious, or otherwise uncooperative. This process is intentionally designed as a "nuclear option": it always liquidates the **entire channel balance**, ensuring a clean and final settlement. The Federation acts as a neutral arbiter, making decisions based on cryptographic proof.

#### Actors
- [BTC Federation](/architecture/common/entities/btc_federation.md) - The arbiter and custodian of L1 funds.
- The User - [vTCP Node](/architecture/common/entities/vtcp_network_node.md) - The party initiating the forced redemption.
- The Hub - [vTCP Hub](/architecture/common/entities/vtcp_network_hub.md) - The unresponsive or disputed counterparty.

#### Process Steps

**Step 1: Unilateral Redemption Request**
- The User initiates a non-cooperative redemption by submitting a request to the Federation.
- This request **must** include:
  1. The most recent [channel reconciliation](/architecture/common/entities/vtcp_channel_reconciliation.md) they possess, which must be signed by both the User and the Hub.
  2. The User's desired L1 BTC destination address.
- *(Security: The co-signed reconciliation is the User's proof of the channel's last agreed-upon state. The Federation will not proceed without it.)*

**Step 2: Federation Verification and Challenge Window**
- The Federation performs initial validation:
  1. It verifies the signatures on the submitted reconciliation. If invalid, the request is rejected.
  2. **It checks the reconciliation's sequence number. If the number is not strictly greater than the sequence number of the state currently on record for that channel, the request is rejected.**
- If validation passes, the Federation initiates a mandatory **Challenge Window**.
- The Federation notifies the Hub that a non-cooperative redemption has been initiated, warning it that failure to respond will result in forced closure based on the User's submitted state.

**Step 3: Hub's Response (Or Lack Thereof)**
- The Hub has two potential courses of action during the Challenge Window:
  - **Option A: No Response.** If the Hub is truly offline or unable to respond, it does nothing.
  - **Option B: Contest the Redemption.** If the Hub believes the User submitted an outdated state, it must submit its own proof: a channel reconciliation with a **higher sequence number**, also co-signed by both parties.
- *(Security: If the Hub contests, the Challenge Window is **reset and restarted** to give the User a full window to respond, ensuring fairness.)*

**Step 4: Final Adjudication**
- At the end of the Challenge Window, the Federation makes a final, binding decision based on the reconciliation with the **highest valid sequence number**.
  - **Scenario 1 (No Hub Response):** The User's submitted state is deemed final.
  - **Scenario 2 (Hub Contests):** If the Hub submitted a reconciliation with a higher sequence number, the Federation accepts the Hub's state as definitive. The party that submitted the outdated state (the User) is flagged and may face a reputation penalty.

**Step 5: Full Channel Liquidation and Settlement**
- Based on the final, adjudicated channel state, the Federation liquidates the entire channel balance.
- **User's Funds:** The User's portion of the balance is sent to their specified L1 BTC address.
- **Hub's Funds:** The Hub's portion of the balance is moved to a special internal holding account within the Federation's registry.
- The Federation updates its registry to permanently close the channel.

### Flow 4: Channel Rebalancing

#### Overview
Rebalancing redistributes settlement line boundaries by reallocating locked funds between multiple channels. This process does not alter the total balance of any channel; it only updates the Federation's registry to reflect how much BTC is locked by each party within those channels. No on-chain (L1) transaction occurs.

#### Trigger & Precursors
Channel Rebalancing can only be initiated by a Hub. The process is typically triggered when a Hub needs to manage liquidity across its channels.

#### Process Steps
**Step 1: Rebalancing Submission**
- The Hub submits a rebalancing request to the Federation, including the signed [channel state reconciliations](/architecture/common/entities/vtcp_channel_reconciliation.md) for all affected channels.
- Each reconciliation must be signed by both the Hub and the corresponding User.

**Step 2: Rebalancing Window and User Verification**
- The Federation validates all signatures. If any are invalid, the request is rejected.
- It then opens a time-limited **Rebalancing Window**.
- The Federation notifies all affected Users that a rebalancing process has been initiated.
- This window allows a User to protect themselves by submitting a more recent signed state if the Hub submitted an outdated one. **The Federation will only accept a user-submitted state if its sequence number is strictly greater than the one submitted by the Hub.**

**Step 3: Federation Decision (Optimistic Consensus)**
- When the window closes, the Federation finalizes the decision based on the reconciliations with the **highest valid sequence numbers**.
- If no User has submitted an updated channel state, the Hub's initial submission is accepted as correct.
- **Offline Users:** The Federation's real-time state backup service protects offline users by ensuring the latest state is always on record.

**Step 4: Registry and L2 Update**
- The Federation updates its internal registry to reflect the new channel boundaries.
- The rebalancing is complete after the Federation confirms the changes and the Hub adjusts the channel boundaries on the vTCP (L2) network.

### Flow 5: Hub Audit

#### Overview
The Hub Audit is a supervisory process initiated by the Federation to verify the integrity and solvency of a Hub's operations. The full, detailed process is defined in the [Proof-of-Reserve Audit Protocol](/architecture/btc-federation/protocols/Proof-of-Reserve%20Audit%20Protocol.md).

#### Key Features
- **Frequency:** Audits are conducted at least **once every 2 hours**, with additional random and event-triggered checks.
- **Mechanism:** The system uses a **Merkle tree-based Proof-of-Reserve (PoR)** model.
- **Public Verifiability:** Users can independently verify their channel balances are included in the Hub's reported liabilities.
- **Reporting:** Audit reports are assigned sequential IDs. Hubs must store full reports for 180 days; the Federation stores the most recent report and all historical root hashes.

## Protocol Constants

The following values are critical for network security and stability. They are set by the Federation and can only be updated via its governance mechanism.

- **L1 Confirmation Threshold:** `6 blocks`
  - **Justification:** A standard for Bitcoin transaction finality, providing a strong defense against block reorganizations and ensuring deposits are irreversible before vTCP tokens are issued.

- **Non-Cooperative Challenge Window:** `72 hours` (432 blocks)
  - **Justification:** This provides a sufficient buffer for a user or Hub to respond to a non-cooperative action, accounting for weekends, holidays, and potential downtime. It is long enough to require deliberation but short enough to prevent funds from being locked indefinitely.

- **Rebalancing Window:** `3 hours` (18 blocks)
  - **Justification:** Rebalancing is a less critical operation than a full channel closure. This shorter window is sufficient for an automated system (or an attentive user) to detect and counter a malicious rebalancing attempt without introducing excessive delays to Hub liquidity management.

- **Maximum Active Deposits per User:** `1`
  - **Justification:** As a primary defense against issuance griefing, a user may only have one in-flight deposit request at any given time.

## Security Considerations

A detailed analysis of potential attacks and their mitigations is available in the following document:
- [BTC <-> vTCP Custody Protocol - Attacks](./BTC%20<->%20vTCP%20Custody%20Protocol%20-%20Attacks.md)

## Future Improvements
- **Deposit to multiple Hubs in batch:** Allow Federation to split the deposit into multiple Hubs from the list of Hubs that has been specified by the user during the deposit request.