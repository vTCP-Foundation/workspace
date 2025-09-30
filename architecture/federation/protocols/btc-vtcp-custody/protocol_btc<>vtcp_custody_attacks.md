# BTC ⟷ vTCP Custody Protocol - Security Analysis and Attack Vectors

_Mykola Ilaschuk, Dima Chizhevsky, 🤖 Gemini 2.5 Pro, 🤖 Claude 4 Sonnet_
<br>
_v0.2, 2025-07-07_
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
    - **Strictly Greater Sequence Number Rule:** As clarified in the redemption protocol v0.2, a contestation is only considered valid if the sequence number of the contesting state is **strictly greater** than the sequence number of the current "winning" state in the redemption attempt. This prevents griefing attacks where a malicious party could reset the challenge window with an equivalent or inferior state.

**1.2. Reversal Fraud (L2/L1 Double Spend)**
- **Attack:** A malicious user attempts to receive L2 funds from the Hub and then fraudulently reclaim their original L1 deposit, effectively stealing from the Hub.
    1. The user completes the deposit, the Hub issues the L2 funds, and the user receives them.
    2. Before the Hub can submit the final `ConfirmL2Issuance` proof to the Federation, the user initiates a `RequestDepositReversal`.
    3. Because the deposit is not yet marked as "settled," the Federation accepts the reversal request and opens the challenge window.
    4. If the Hub fails to contest within this window, the Federation returns the L1 funds to the user, who has now successfully claimed both L1 and L2 funds.
- **Mitigation:**
    - **Hub Liveness Requirement:** This attack is fundamentally a race condition. The protocol's primary defense is the Hub's ability to contest the fraudulent reversal. This imposes a strict liveness requirement on the Hub, which must monitor for reversal requests and respond immediately with the `ContestDepositReversal` call, providing the co-signed `ChannelReconciliation` as proof of L2 issuance.
    - **Extended Challenge Window:** The `DEPOSIT_REVERSAL_CHALLENGE_PERIOD_SECONDS` is set to 72 hours. This period is intentionally long and aligned with the standard redemption window. The rationale is that Hubs are heavily invested, professional entities. A failure to issue L2 funds is considered a less likely fault scenario than a user attempting to redeem. This extended window gives a Hub ample time to recover from any temporary outage and contest a fraudulent reversal, prioritizing the Hub's security against this form of theft. At the same time, it provides a reliable, albeit slow, recovery path for users in the case of a genuinely failed Hub.
    - **Proactive L2 Issuance Confirmation:** The most critical defense for the Hub is to call `ConfirmL2Issuance` immediately after the L2 funds are issued and co-signed. This action marks the deposit as "settled" and permanently prevents any future reversal attempts for that `deposit_id`. It is also recommended that the user's client calls this endpoint as well for redundancy.

**1.3. Hub Resource Liveness Attack (Issuance Griefing)**
- **Attack:** A malicious user repeatedly initiates the issuance process but intentionally abandons it before committing resources. In the original protocol design, this forced the Hub to reserve capital and a channel slot for a request that would never be fulfilled, allowing an attacker to cheaply exhaust a Hub's resources and deny service to legitimate users.

- **Mitigation: Hub Attestation Flow**
    - The protocol was updated to completely mitigate this vector by shifting the burden of proof onto the user *before* the Federation engages any Hub resources. The new flow introduces a mandatory **Hub Attestation** step.
    - **1. Pre-authentication:** The user must first connect directly to a Hub and satisfy its individual policies (e.g., KYC, CAPTCHA, rate-limiting). If successful, the Hub provides a signed, single-use `attestation_token`.
    - **2. Attestation in Request:** The user includes this token in their `DepositRequest` to the Federation.
    - **3. Federation Verification:** The Federation now acts as a verifier. It contacts the Hub and asks, "Did you issue this token?" The Hub validates its own token, and only if it is valid does the Hub commit its resources and the protocol proceeds.
    - **Result:** A Hub's resources are never reserved unless it has explicitly pre-approved the user. This makes the griefing attack impossible, as the Hub can simply refuse to issue attestations to clients that fail its policy checks.

- **Visualizing the Mitigation:**

    ```mermaid
    graph TD
        subgraph Old Flow (Vulnerable)
            A[User] -->|1. Request(preferred_hub)| B(Federation);
            B -->|2. Select Hub, Publish Auth| C{Hub};
            C -->|3. Reserve Resources| C;
            A -->|4. Abandon| D((Stop));
        end

        subgraph New Flow (Mitigated)
            X[User] -->|1. Request Attestation| Y{Hub};
            Y -->|2. Issue Token| X;
            X -->|3. Request(attestation_token)| Z(Federation);
            Z -->|4. Verify Token w/ Hub| Y;
            Y -->|5. Accept & Reserve Resources| Y;
        end

        style C fill:#f9f,stroke:#333,stroke-width:2px
        style Y fill:#9cf,stroke:#333,stroke-width:2px
    ```


**1.4. ECDSA Signature Malleability**
- **Attack:** Without enforcing canonical low-S form or Schnorr-style deterministic signatures, an attacker can produce many valid signatures for the exact same payload. This could lead to multiple `deposit_id` hashes pointing to the same L1 transaction, complicating audits, or bypassing simple replay defenses that key only on the signature bytes.
- **Mitigation:**
    - **Canonical Signatures:** As specified in [protocol_btc<>vtcp_custody_deposit.md#hashing-and-signing-algorithm](/architecture/federation/protocols/protocol_btc<>vtcp_custody_deposit.md#hashing-and-signing-algorithm), user signatures for `DepositRequest` MUST be in canonical low-S form and DER-encoded. This ensures a unique signature representation for each valid payload, preventing malleability.
    - **Consider BIP-340 Schnorr:** For future enhancements, consider adopting BIP-340 Schnorr signatures for users, which inherently avoids malleability and allows for clean public key recovery.

**1.5. Hub Preference List Manipulation (Authorization Expiry Extension)**
- **Attack:** A malicious user crafts a `DepositRequest` with an excessive number of Hub attestations (many of which may be invalid or for unresponsive Hubs) to artificially extend the deposit authorization expiry time. Since the expiry is calculated as `base_time + (num_hubs * per_hub_time)`, including many Hubs gives the attacker a longer window to complete the deposit, potentially tying up Federation and Hub resources for extended periods.
- **Mitigation:**
    - **Maximum Hub Limit:** As specified in [protocol_btc<>vtcp_custody_deposit.md#protocol-constants](/architecture/federation/protocols/protocol_btc<>vtcp_custody_deposit.md#protocol-constants), the protocol enforces `MAX_HUBS_PER_REQUEST = 5`, limiting the number of Hubs that can be specified in a single request. This caps the maximum authorization expiry time and prevents resource lock extension attacks.
    - **Early Validation:** The Federation validates the Hub count limit during the initial `DepositRequest` processing, immediately rejecting requests that exceed the limit before any expensive operations (Hub verification, resource allocation) are performed.
    - **Hub Reputation Tracking:** The Federation should maintain reputation scores for Hubs to detect patterns of invalid attestations and potentially prioritize reputable Hubs in the verification process.

**1.6. DoS via Request Flooding**
- **Attack:** An external actor can spam thousands of bogus `DepositRequests` (e.g., with invalid signatures or malformed data), consuming Federation resources (CPU for signature checks, network bandwidth, storage for logging) and potentially disrupting legitimate service.
- **Mitigation:**
    - **Lightweight Pre-filters:** Implement a multi-layered pre-filtering mechanism before expensive cryptographic operations (hashing, signature verification).
        - **Rate-Limiting:** Apply rate-limiting per IP address or per `vID` (after initial parsing) to restrict the number of requests within a given time window.
        - **Proof-of-Work (PoW):** Introduce a client-side Proof-of-Work challenge for `DepositRequest` submissions. The complexity of the PoW can be dynamically adjusted, potentially increasing for IP addresses or IP zones identified as sources of excessive or invalid requests. This imposes a computational cost on attackers, making large-scale spamming economically unfeasible.
    - **Early Rejection of Malformed Protobufs:** Implement strict and fast validation of incoming message formats. Malformed protobufs or requests that fail basic structural checks should be rejected immediately, without proceeding to more resource-intensive steps like hashing or signature verification.

**1.7. DoS via Redemption Transaction Log Spam**
- **Attack:** During a non-cooperative redemption, a malicious party submits a `RedemptionRequest` with a valid base state but includes an enormous list of subsequent transactions in the `subsequent_transactions` field. This forces the Federation to perform excessive computation to validate each transaction and calculate the "effective state," potentially overwhelming its resources and delaying legitimate redemptions for other users.
- **Mitigation:**
    - **Transaction Limit:** As specified in the redemption protocol v0.2, the Federation enforces a `MAX_SUBSEQUENT_TRANSACTIONS_PER_REQUEST` limit (defaulting to 512). Any `RedemptionRequest` exceeding this limit is rejected immediately, preventing the computational DoS attack. This also implies a requirement on the vTCP protocol to ensure that channels do not accumulate more than this number of un-reconciled transactions.

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

#### **Category 3: Economic and Collusion Attacks**

This category involves multiple parties colluding or exploiting economic incentives to undermine the system.

**3.1. Federation Resource Drain via Unspecified Redemption Fees**
- **Attack:** The protocol for redemption does not explicitly state how L1 transaction fees for the final payout are handled. If the Federation were to cover this fee from its own operational funds, an attacker could repeatedly open small channels, redeem them, and force the Federation to pay the L1 fees, creating a slow but steady drain on its resources.
- **Mitigation:**
    - **Self-Funding Redemptions:** The redemption protocol v0.2 explicitly states that the L1 transaction fee for the redemption payout is deducted from the final balances of the User and the Hub, proportional to their share of the funds. This makes redemptions self-funding and removes the economic incentive for this attack.

**3.2. User-Hub Collusion to Defraud Third Parties**
- **Attack:** A user and a Hub collude to create a fraudulent channel state showing a very large balance. They then present this fake state to an external system (e.g., a DeFi lending platform) as proof of funds to secure a loan they have no intention of repaying.
- **Mitigation:**
    - **Federation as the Single Source of Truth:** The protocol establishes the Federation as the sole arbiter of channel states. Any external entity wishing to verify a channel's balance **must** query the Federation's public API or validate the state against the published Proof-of-Reserve audit data (Flow 5). An off-chain state provided by a user and Hub alone must never be trusted.
    - **Public Audit Trail (Flow 5):** The regular, public, Merkle-based audits allow any third party to independently verify a Hub's solvency and, with the user's help, confirm a specific channel's inclusion in the audit.

**3.3. Federation Member Collusion (Hub Stake Overflow)**
- **Attack:** A malicious supermajority of Federation members collude to approve vTCP token issuance for a Hub without a corresponding L1 deposit, creating unbacked vTCP tokens. This is the "Hub Stake Overflow" scenario.
- **Mitigation:**
    - **Decentralization and M-of-N Trust Model:** The fundamental security of the entire system rests on the assumption that a qualified majority (M) of the N Federation members are honest. The primary mitigation is ensuring that the N members are operated by distinct, non-colluding entities in different legal and geographical jurisdictions.
    - **Radical Transparency and Public Auditing:** All Federation decisions (issuance approvals, dispute resolutions) and the evidence used (L1 transaction IDs, signed reconciliations) must be published to a public, immutable log. This allows independent, third-party auditors to continuously verify that the Federation is adhering to the protocol rules. If a colluding majority approves an issuance without a corresponding L1 deposit, the fraud will be immediately visible in the public record.
