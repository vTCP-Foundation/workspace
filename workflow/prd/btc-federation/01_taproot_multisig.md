# Project Requirements Document (PRD)

## Document Information
- **Project Name**: BTC Federation
- **Phase/Iteration**: Phase 1
- **Document Version**: 1.0
- **Date**: 2025-06-03
- **Author(s)**: Dima Chizhevsky
- **Stakeholders**: Dima Chizhevsky, Mykola Ilashchuk
- **Status**: Active
- **Related Documents**: [BTC FastNet Local Node Setup](workspace/workflow/tasks/btc-federation/0/00_btc_fastnet_local_node_setup.md)

## Executive Summary
**Objective**: Validate taproot multisig scalability for BTC federation with up to 512 participants using 50% threshold signing.

**Scope**: Performance testing of MuSig2 and FROST schemes across different participant counts to determine practical federation limits.

## Problem Statement
### Background
BTC federation requires distributed transaction validation and signing across 256-512 nodes. We need to validate if taproot's Schnorr signature schemes (MuSig2 and FROST) can support this scale efficiently with 50% threshold requirements.

### Success Metrics
- **Primary**: Complete multisig workflow including blockchain transactions within 2 minutes for all tested participant counts:
  - Multisig operations (address creation, key generation, message signing, signature verification)
  - **Transaction 1**: Miner to multisig address transfer with 1 confirmation
  - **Transaction 2**: Multisig to miner address transfer with 1 confirmation
  - Balance verification after each transaction
  - Transaction hash and balance logging
  - **Note**: Blockchain transaction execution times are excluded from multisig performance metrics
- **Secondary**: Performance benchmarks collected for 64, 128, 256, and 512 participants

## Project Scope
### Deliverables
- Golang test suite measuring taproot multisig performance
- Performance report with key metrics per participant count
- Documentation of practical participant limits and performance metrics

### Test Scenarios
- **Threshold Configurations**: M-of-N where M = 50% of N
  - 32-of-64, 64-of-128, 128-of-256, 256-of-512 participants
- **Schemes**: Both MuSig2 and FROST implementations
- **Operations**: 
  - Key generation, transaction signing, signature verification
  - **Blockchain Transaction Workflow**:
    - Create multisig address
    - Execute Transaction 1: Miner → Multisig (wait for 1 confirmation)
    - Verify multisig balance increase and miner balance decrease
    - Execute Transaction 2: Multisig → Miner (wait for 1 confirmation)  
    - Verify multisig balance decrease and miner balance increase
    - Log all transaction hashes and balance changes
- **Environment**: Local regtest network (BTC FastNet setup)

### Out of Scope
- Mainnet/testnet testing
- BTC network propagation timing (beyond 1 confirmation requirement)
- Byzantine fault tolerance testing

## Technical Requirements
- **Language**: Go
- **Environment**: Local Bitcoin regtest node
- **Schemes**: MuSig2 and FROST taproot implementations
- **Dependencies**: BTC FastNet Local Node Setup completion
- **Metrics**: Operation timing (p2p coordination included), success rates, resource usage
- **Blockchain Integration**:
  - Transaction broadcasting and confirmation monitoring
  - Balance tracking for miner and multisig addresses
  - Transaction hash and balance logging
  - 1-confirmation validation for each transaction

### Simulation Methodology
- **Participant Simulation**: Parallel execution limited by available CPU cores
- **Core Detection**: Automatically determine available cores and run maximum parallel operations
- **Metric Adjustment**: Scale results from `cores_used` to full participant count (e.g., 8 cores → 512 participants)
- **Latency Simulation**: Add 200ms inter-participant latency during result representation phase
- **Validation**: Compare adjusted metrics against 2-minute threshold per operation
- **Blockchain Integration**: Execute actual blockchain transactions for functional validation without including transaction times in multisig performance measurements

### Performance Metrics
- **Timing**: Operation duration (adjusted for participant count)
- **Computational Complexity**: CPU cycles, algorithmic complexity analysis
- **Memory Usage**: RAM consumption per participant, scaling characteristics
- **Resource Scaling**: Linear vs non-linear scaling identification
- **Blockchain Metrics** (for functional validation only):
  - Transaction confirmation time (1 confirmation required)
  - Balance verification accuracy
  - Transaction hash logging and validation
  - End-to-end workflow completion time (including blockchain operations)
- **Metric Separation**: Multisig cryptographic operation performance is measured independently from blockchain transaction execution times

## Risk Assessment
- **High Risk**: 256-of-512 participant threshold may exceed practical limits
- **Medium Risk**: Core-limited simulation may not capture all distributed system bottlenecks
- **Low Risk**: 200ms latency addition may underestimate real network conditions
- **Mitigation**: Test incrementally starting with smaller thresholds; establish fallback participant limits; document scaling assumptions and simulation limitations

## Task List
**Reference**: [Task List for PRD 01](../../tasks/btc-federation/1/tasks.md)

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.1 | 2025-06-03 | Dima Chizhevsky | Initial draft for Phase 1 | Phase 1 |