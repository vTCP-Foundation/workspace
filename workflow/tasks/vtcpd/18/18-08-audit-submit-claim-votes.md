# 18-08 - Audit Completion Claim Votes Submission

# Links
- [PRD-18: Audit Mechanism Based on Finalized Transactions](../../prd/vtcpd/18-audit-mechanism-finalized-transactions.md)
- [Previous task: 18-04-audit-source-transaction](18-04-audit-source-transaction.md)
- [Previous task: 18-05-audit-target-transaction](18-05-audit-target-transaction.md)
- [Previous task: 18-06-other-audit-transactions](18-06-other-audit-transactions.md)

# Description

This task adds fire-and-forget SubmitClaimVotes RPC submissions before successful audit completion.
If the local node has finalized transactions that are not finalized on the counterparty side,
then the final set of participant signatures must be submitted to the observer. The requests
must be sent before `trustLineActionSignal` and must not delay or block audit completion.

The changes apply to all four audit transactions:
- `AuditSourceTransaction`
- `AuditTargetTransaction`
- `SetOutgoingTrustLineTransaction`
- `CloseIncomingTrustLineTransaction`

# Requirements and DOD

## Requirements

1. **Identify asymmetrically finalized transactions**
   - Track transactions finalized locally but not finalized on the counterparty.
   - For initiator-side transactions (AuditSource, SetOutgoing, CloseIncoming):
     - Use the UUIDs from `Audit_UpdateTransactionsList` received from the counterparty as the
       set of locally finalized but not finalized remotely.
   - For contractor-side transaction (AuditTarget):
     - Use the set of locally finalized transactions that were excluded because they were
       missing in the initiator's list ("contractor has extra transactions" case).

2. **Prepare SubmitClaimVotes data**
   - Load participant signatures from `paymentParticipantsVotesHandler()->participantsSignatures(uuid)`.
   - Load the own payment public key using `paymentKeysHandler()->getOwnPublicKey()`.
   - Load `maximal_claiming_block_number` for each transaction (add handler method if missing).
   - Serialize data using `serializeSubmitClaimVotesForSigning(...)` and sign with
     `Keystore::signPaymentTransaction(...)`.

3. **Submit votes**
   - Send `sendRpcRequest(make_shared<SubmitClaimVotesRpcRequest>(...))` for each UUID.
   - Do not wait for responses, do not transition to wait stage.
   - If signatures or metadata are missing, log a warning and continue.

4. **Ordering and timing**
   - Submit votes only after the audit has been verified and committed locally.
   - Send requests before `trustLineActionSignal` in each transaction.

5. **No functional regression**
   - Audit success path must remain unchanged aside from the new RPC submissions.
   - Failures in vote submission must not affect audit completion.

## Definition of Done

- [ ] Asymmetric finalized transaction list is tracked for each audit transaction
- [ ] SubmitClaimVotes RPCs are sent before `trustLineActionSignal` on success
- [ ] Requests are fire-and-forget (no waiting for responses)
- [ ] Missing signatures or metadata are logged and skipped
- [ ] `maximal_claiming_block_number` lookup implemented if not already available
- [ ] Code compiles without errors

# Implementation Plan

## Step 1: Analyze Existing Code
1. Review success paths in:
   - `src/core/transactions/transactions/trust_lines/AuditSourceTransaction.cpp`
   - `src/core/transactions/transactions/trust_lines/AuditTargetTransaction.cpp`
   - `src/core/transactions/transactions/trust_lines/SetOutgoingTrustLineTransaction.cpp`
   - `src/core/transactions/transactions/trust_lines/CloseIncomingTrustLineTransaction.cpp`
2. Review `CompletedPaymentsObserverMonitoringTransaction::runProcessClaimStatusesStage()`
   for `SubmitClaimVotesRpcRequest` construction and signing.

## Step 2: Add Data Access for Max Claim Block Number
1. If missing, add `maximalClaimingBlockNumber(const TransactionUUID &)` to
   `PaymentTransactionsHandler` and its SQLite/PostgreSQL implementations.
2. Add SQL: `SELECT maximal_claiming_block_number FROM payment_transactions WHERE uuid = ?`.

## Step 3: Track Asymmetric Transactions
1. Add member storage in each audit transaction for UUIDs that are finalized locally but not
   finalized on the counterparty (e.g., `mAsymmetricFinalizedTransactions`).
2. Populate this list when handling:
   - `Audit_UpdateTransactionsList` on initiator side
   - Extra finalized transactions on contractor side
3. Preserve the list across retry and until successful completion.

## Step 4: Implement SubmitClaimVotes Helper
1. Add a helper method (preferably in `BaseTrustLineTransaction`) to:
   - Load own payment public key
   - Load max claim block number
   - Load participant signatures
   - Serialize and sign SubmitClaimVotes payload
   - Send `SubmitClaimVotesRpcRequest`
2. Ensure the helper is safe to call with an empty list.

## Step 5: Wire into Success Paths
1. Call the helper right before `trustLineActionSignal` in each of the 4 transactions.
2. Ensure failures are logged but do not change the audit result.

# Test Plan

**Complexity Level:** Moderate

## Functional Validation
- Verify SubmitClaimVotes is sent for transactions finalized locally but not on counterparty
- Verify no SubmitClaimVotes is sent when the list is empty
- Verify audit completion succeeds even if vote submission fails
- Verify ordering: RPCs are sent before `trustLineActionSignal`

## Integration Validation
- Verify payment public key and signatures are loaded correctly
- Verify correct `maximal_claiming_block_number` is used in SubmitClaimVotes

# Verification and Validation

## Architecture integrity
- Reuses existing SubmitClaimVotes flow patterns from monitoring transaction
- Does not alter audit state machine beyond additional RPC submissions

## Security
- Signed SubmitClaimVotes payload uses existing Keystore signing path
- No new sensitive data exposure

## Performance
- Fire-and-forget submissions avoid blocking audit completion
- Submission only for mismatched finalized transactions

## Scalability
- Submissions are per-transaction; expected count is low

## Reliability
- Failures in vote submission do not prevent audit success
- Missing data is logged and skipped

## Maintainability
- Helper method centralizes submission logic to avoid duplication

## Cost
- N/A

## Compliance
- N/A

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
