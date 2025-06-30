# BTC ⟷ vTCP Custody Protocol - Security Analysis and Attack Vectors

_Mykola Ilaschuk, Dima Chizhevsky, 🤖 Gemini 2.5 Pro, 🤖 Claude 4 Sonnet_
<br>
_v0.1, 2025-06-30_
<br>
<br>

This document provides a detailed analysis of potential attacks against the [BTC ⟷ vTCP Custody Protocol](BTC%20<->%20vTCP%20Custody%20Protocol.md), categorized by the malicious actor, and the specific protocol mechanisms designed to mitigate them.

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
