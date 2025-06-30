# Proof-of-Reserve (PoR) Audit Protocol

_Mykola Ilaschuk, Dima Chizhevsky, Gemini 2.5 Pro, Claude 4 Sonnet_
<br>
_v0.1, 2025-06-30_
<br>
<br>

## Overview
This protocol defines the mechanism by which the Federation audits Hubs to ensure their solvency and operational integrity. It uses a Merkle tree-based Proof-of-Reserve (PoR) system to efficiently verify a Hub's liabilities without exposing sensitive user data publicly. This process is designed to be regular, unpredictable, and publicly verifiable.

## Core Principles
- **Solvency Guarantee:** To prove that the Hub's total L2 liabilities are fully backed by its L1 stake.
- **Efficiency:** To use cryptographic commitments (Merkle roots) to avoid transmitting large datasets for every audit.
- **Public Verifiability:** To allow any user to independently verify that their channel balance is correctly included in the Hub's reported liabilities.
- **Unpredictability:** To use random checks to keep Hubs honest, as they can be audited at any moment.

## Actors
- **BTC Federation:** The auditor.
- **The Hub:** The auditee.
- **The User:** An independent verifier of their own funds.

## Audit Process

### Step 1: Audit Trigger
The Federation audits every Hub under its supervision according to the following triggers:
1.  **Scheduled Audit:** A mandatory audit is performed at least **once every 2 hours**.
2.  **Randomized Audit:** The Federation can and will issue random, unpredictable checks at any time. These checks can be for the entire Hub or a randomly selected subset of its channels.
3.  **Event-Based Audit:** An audit can be triggered by risk flags, as described in the main protocol document.

### Step 2: Merkle Root Submission
- Upon receiving an audit request, the Hub must construct a **Merkle tree** of all its active settlement channel states.
- Each leaf in the tree represents a single channel and is a hash of the following data: `hash(channel_id + user_id + hub_id + user_balance + hub_balance + nonce)`.
- The Hub submits the final **Merkle root** to the Federation. This root is a cryptographic commitment to the Hub's total liabilities at that specific moment.

### Step 3: Federation Verification
- The Federation demands that the Hub reveal the data for a randomly selected subset of the Merkle tree's leaves.
- The Federation verifies that the revealed leaves correctly hash to the provided Merkle root.
- It cross-references the data in the revealed leaves with its own internal records to check for consistency.
- The Federation calculates the Hub's total liability (which the Hub must also provide) and verifies it against the Hub's L1 stake.

### Step 4: Public Proof and User Verification
- Upon successful verification, the Federation signs the Merkle root and publishes it to a public, well-known location.
- The Hub is required to make the full Merkle tree data available to its users.
- A User's node can then:
  1. Request its specific leaf data and the corresponding Merkle proof (the branch of hashes leading to the root) from the Hub.
  2. Independently calculate the Merkle root using the proof.
  3. Compare its calculated root with the publicly signed root from the Federation.
- If the roots do not match, or if the user's balance is incorrect, the user's node can automatically initiate a dispute with the Federation.

## Audit Reporting and Storage

### Audit ID
- Each audit report is assigned a **monotonically increasing sequential number** as a unique identifier.

### Storage Requirements
- **The Hub:** Is required to store the full audit reports (including the complete Merkle tree data) for the last **180 days**.
- **The Federation:** For efficiency, the Federation only stores the *full report* of the most recent audit. For all previous audits, it retains only the `audit_id` and the corresponding signed `merkle_root`.
- **On-Demand Publication:** The Federation can, at any time, demand that a Hub publish a specific historical audit report (from within its 180-day window) for public inspection. Failure to comply results in severe penalties.
