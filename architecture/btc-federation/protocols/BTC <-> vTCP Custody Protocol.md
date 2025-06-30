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
This flow enables a user to lock BTC on L1 and receive an equivalent amount of vTCP tokens on L2, backed by the locked BTC. The process is designed to be secure against malicious actors and to handle non-cooperative scenarios gracefully.

#### Actors
- [BTC Federation](/architecture/common/entities/btc_federation.md) - Manages the custody protocol
- The Hub - [vTCP Hub](/architecture/common/entities/vtcp_network_hub.md) - Provides L2 connectivity
- The User - [vTCP Node](/architecture/common/entities/vtcp_network_node.md) - Initiates deposit

#### Process Steps

**Step 1: Signed Deposit Request & Authorization**
- The User submits a **digitally signed** deposit request to the Federation, specifying the amount and preferred Hub.
- The Federation verifies the user's on-chain funds and checks the Hub's capacity. If valid, it authorizes the request by generating a unique L1 multisig address for the deposit.
- The Federation sends a signed authorization message to the Hub, which includes the unique L1 deposit address.
- *(Security: The Federation processes only one active request per user address at a time. The initial signature proves ownership and allows for fund verification.)*

**Step 2: L2 Channel Establishment & Address Relay**
- The Hub receives the authorization and establishes a [Settlement Line](/architecture/common/entities/vtcp_settlement_line.md) with the User's node.
- As part of the channel handshake, both the Hub and User sign the initial zero-balance channel reconciliation. This proves the L2 channel was successfully created.
- The Hub then relays the unique L1 deposit address (received from the Federation) to the User.
- *(Security: This step acts as a liveness check. If the L2 channel cannot be established, the process halts before any funds are moved.)*

**Step 3: L1 Deposit**
- The User receives the L1 address from the Hub and deposits the specified BTC amount.
- **This action serves as the implicit success signal to the Federation.** By monitoring the deposit address it created, the Federation knows the L2 channel was established and the user is proceeding. No direct notification from the Hub is required.

**Step 4: L2 Token Issuance**
- The Federation waits for a mandatory number of block confirmations on the user's deposit.
- Once confirmed, the Federation sends a final confirmation to the Hub.
- The Hub issues the equivalent vTCP tokens to the User through the established L2 channel.

**Step 5: Handling Failures and Abandonment**
- **Case 1: L2 Channel Fails:** If the Hub and User cannot establish the L2 channel (Step 2), the reservation window simply expires. Since no initial reconciliation was signed, there is no proof of who was at fault. Neither party's reputation is affected.
- **Case 2: User Abandons Deposit:** If the L2 channel is created but the User fails to deposit funds (Step 3), the Hub can protect its reputation. It submits the initial, zero-balance reconciliation (signed by both parties) to the Federation. This proves the Hub fulfilled its duty and the User was responsible for the abandonment. The Federation then lowers the User's reputation score.
- **Case 3: Hub Fails to Issue vTCP Tokens:** If the User deposits funds but the Hub fails to issue tokens (Step 4), the User opens a dispute with the Federation as described in the standard dispute process, providing the L1 transaction hash as proof.

### Flow 2: Cooperative Redemption

#### Overview
This flow enables a User to cooperatively redeem their vTCP tokens for an equivalent amount of BTC on L1. The process is designed to be fast and efficient, relying on a joint, cryptographically-signed agreement between the User and the Hub. This mutual consent eliminates the need for lengthy objection windows and ensures the Federation can act on the request with confidence.

#### Actors
- [BTC Federation](/architecture/common/entities/btc_federation.md) - Manages the custody protocol and L1 funds.
- The Hub - [vTCP Hub](/architecture/common/entities/vtcp_network_hub.md) - The User's L2 counterparty.
- The User - [vTCP Node](/architecture/common/entities/vtcp_network_node.md) - Initiates the redemption.

#### Process Steps

**Step 1: Redemption Agreement & State Reconciliation**
- The User signals their intent to redeem a specific amount of vTCP tokens to the Hub.
- The User and Hub collaborate to create and co-sign a new [channel reconciliation](/architecture/common/entities/vtcp_channel_reconciliation.md). This reconciliation reflects the new channel state *after* the redemption amount has been deducted from the User's balance and effectively transferred to the Hub's side of the channel on L2.
- *(Security: This co-signed state is the definitive, non-repudiable proof of the agreement. It prevents either party from later disputing the amount.)*

**Step 2: Joint Withdrawal Request**
- The User submits the co-signed channel reconciliation to the Federation, along with their desired L1 BTC destination address.
- As part of the same request, the Hub must provide a confirmation signature, attesting to the validity of the redemption.
- *(Efficiency: This joint submission provides the Federation with immediate proof of consent from both parties, making the cooperative process significantly faster.)*

**Step 3: Federation Verification**
- The Federation receives the joint request and performs the following critical checks:
  1. Verifies the signatures from both the User and the Hub on the channel reconciliation.
  2. **Checks the reconciliation's sequence number. It MUST be strictly greater than the sequence number of the state currently on record for that channel. If not, the request is rejected.**
  3. Compares the submitted reconciliation with its own internal record of the channel's state to ensure it is current and the user has sufficient funds.
- If any verification fails, the Federation rejects the request and notifies both parties.

**Step 4: L1 Fund Release & L2 State Update**
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

### Attack Vectors

#### Hub Stake Overflow
A Hub Stake Overflow is a critical failure state where a Hub manages to issue more vTCP tokens than it has backing for in its L1 BTC stake. This would render the Hub insolvent and break the fundamental 1:1 promise of the protocol. Preventing this scenario is a primary security goal.

##### Potential Causes:
1.  **Software Exploit:** A bug in the Federation's consensus or validation logic could be exploited by a malicious Hub to create unbacked vTCP tokens.
2.  **Collusion:** A malicious supermajority of Federation members could collude to approve issuance for a Hub without a corresponding L1 deposit.
3.  **Replay Attacks:** An attacker could attempt to re-submit a valid, historical deposit transaction to trick the Federation into crediting funds twice.

##### Mitigation Strategies:
- **Strict Federation Consensus:** The core protocol enforces that no single Federation member can approve issuance. Any L2 crediting event requires cryptographic verification of an on-chain L1 transaction, validated independently by all members of the Federation's consensus group.
- **Mandatory Hub Audits:** The `Flow 5: Hub Audit` serves as a direct and powerful mitigation. Regular and event-triggered audits ensure that any overflow state is detected and contained quickly.
- **Unique Deposit Addresses:** As defined in `Flow 1`, every new issuance request generates a unique, single-use L1 deposit address. This makes replay attacks impossible, as an address is retired immediately after the first successful deposit.
- **Anomaly Detection:** The Federation is expected to run monitoring software to detect unusual issuance patterns (e.g., abnormally high velocity or volume from a single Hub), which can trigger an automatic audit.

### Advanced Security Analysis and Attack Vectors

This section provides a detailed analysis of potential attacks, categorized by the malicious actor, and the specific protocol mechanisms designed to mitigate them.

---

#### **Category 1: Malicious User Attacks**

A malicious user may attempt to defraud the Hub or disrupt the network.

**1.1. Outdated State Redemption Attack**
- **Attack:** During a Non-Cooperative Redemption (Flow 3), a user submits a historical channel state where they had a higher balance, attempting to steal funds from the Hub.
- **Mitigation:**
    - **Nonce/Sequence Number:** Every channel reconciliation has a monotonically increasing sequence number. The Federation will always reject a state if a state with a higher number has already been seen.
    - **Hub Challenge Window (Flow 3, Step 3):** The Hub is given a mandatory window to contest the user's claim by providing its own channel state. The state with the highest sequence number always wins.
    - **Window Refresh Logic (Flow 3, Step 3 Security Note):** If the Hub successfully contests the claim, the challenge window is reset and restarted. This prevents the user from waiting until the last second to submit their outdated state, giving the Hub no time to react.

**1.2. Issuance Griefing Attack**
- **Attack:** A user repeatedly initiates the issuance process (Flow 1, Steps 1-2) but intentionally abandons it before depositing BTC. This forces the Hub to expend resources (opening L2 channels, reserving capital) for no reason.
- **Mitigation:**
    - **One Active Deposit Limit:** The protocol strictly enforces that a user can only have one active deposit request at a time (see Protocol Constants).
    - **Reputation System (Flow 1, Step 5):** If a user abandons a deposit after the L2 channel is established, the Hub can submit the co-signed, zero-balance initial reconciliation. This serves as cryptographic proof of the user's failure to complete the process, leading to a reputation penalty.
    - **Economic Deterrents (Proposed):** For users with low reputation scores or for high-value issuance requests, the Federation could require a small, refundable BTC bond to initiate the process. The bond is forfeited if the user abandons the deposit, creating a direct economic disincentive for griefing.

---

#### **Category 2: Malicious Hub Attacks**

A malicious Hub may attempt to steal user funds, create unbacked vTCP tokens, or deny service.

**2.1. Outdated State Challenge Attack**
- **Attack:** A user initiates a legitimate Non-Cooperative Redemption (Flow 3). The malicious Hub, which is online, contests the user's valid claim by submitting an older channel state where the user's balance was lower.
- **Mitigation:**
    - **Symmetric Nonce Rule:** This attack is the inverse of the "Outdated State Redemption Attack" (1.1) and is defeated by the same mechanism. The Federation adjudicates based on the highest sequence number, which the honest user can always provide. The "window refresh" logic ensures the user has time to respond to the Hub's false claim.

**2.2. Malicious Rebalancing with Offline User**
- **Attack:** A Hub initiates a Channel Rebalancing (Flow 4) using an outdated state for a specific user's channel, hoping the user is offline and will not be able to contest the fraudulent state within the rebalancing window.
- **Mitigation:**
    - **Balance Floor Guarantee:** The protocol strictly forbids any rebalancing operation from setting a user's channel boundary to a value lower than their current confirmed balance in the latest valid reconciliation. This makes it impossible for a Hub to directly steal funds via this vector.
    - **User Verification Window (Flow 4, Step 2):** For an online user, the rebalancing window provides a direct opportunity to detect and reject a fraudulent rebalancing attempt by submitting their more recent state.

**2.3. L2 Payment Execution Failure**
- **Attack:** A user and Hub engage in an L2 payment. The user sends their payment, but the Hub fails to send its counter-payment and immediately attempts to close the channel (via Flow 3) using the pre-transaction state.
- **Mitigation:**
    - **Atomic Reconciliation Mandate:** The protocol must enforce that L2 payments are only considered final once a new channel reconciliation reflecting the post-payment state is co-signed by both parties. Any L2 client software must be designed to obtain the new signed state *before* releasing its own funds in a multi-part exchange. Failure to secure the updated signature places the fault on the party that sent funds prematurely.

---

#### **Category 3: Collusion Attacks**

This category involves multiple parties colluding to undermine the system.

**3.1. User-Hub Collusion to Defraud Third Parties**
- **Attack:** A user and a Hub collude to create a fraudulent channel state showing a very large balance. They then present this fake state to an external system (e.g., a DeFi lending platform) as proof of funds to secure a loan they have no intention of repaying.
- **Mitigation:**
    - **Federation as the Single Source of Truth:** The protocol establishes the Federation as the sole arbiter of channel states. Any external entity wishing to verify a channel's balance **must** query the Federation's public API or validate the state against the published Proof-of-Reserve audit data (Flow 5). An off-chain state provided by a user and Hub alone must never be trusted.
    - **Public Audit Trail (Flow 5):** The regular, public, Merkle-based audits allow any third party to independently verify a Hub's solvency and, with the user's help, confirm a specific channel's inclusion in the audit.

**3.2. Federation Member Collusion (Hub Stake Overflow)**
- **Attack:** A malicious supermajority of Federation members collude to approve vTCP token issuance for a Hub without a corresponding L1 deposit, creating unbacked vTCP tokens. This is the "Hub Stake Overflow" scenario.
- **Mitigation:**
    - **Decentralization and M-of-N Trust Model:** The fundamental security of the entire system rests on the assumption that a qualified majority (M) of the N Federation members are honest. The primary mitigation is ensuring that the N members are operated by distinct, non-colluding entities in different legal and geographical jurisdictions.
    - **Radical Transparency and Public Auditing:** All Federation decisions (issuance approvals, dispute resolutions) and the evidence used (L1 transaction IDs, signed reconciliations) must be published to a public, immutable log. This allows independent, third-party auditors to continuously verify that the Federation is adhering to the protocol rules. If a colluding majority approves an issuance without a corresponding L1 deposit, the fraud will be immediately visible in the public record.