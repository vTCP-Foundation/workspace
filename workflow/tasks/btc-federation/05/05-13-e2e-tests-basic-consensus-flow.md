# 05-13 - End-to-End Tests for Basic Consensus Flow

# Links
- [PRD](/workflow/prd/btc-federation/05_hotstuff_consensus.md)
- [Implementation Task 05-06](/workflow/tasks/btc-federation/05/05-06-basic-consensus-flow.md)

# Description
Develop comprehensive end-to-end tests for the basic consensus flow implemented in Task 05-06, focusing on complete consensus rounds, multi-node scenarios, and happy path validation.

# Requirements and DOD
- **E2E Coverage**: Complete consensus rounds from proposal to commitment
- **Multi-Node Tests**: 5-node network scenarios with leader rotation
- **Happy Path Focus**: Normal operation without Byzantine behavior
- **Block Progression**: Multiple consensus rounds and chain growth
- **State Consistency**: Validation of consistent state across all nodes
- **Performance Validation**: Consensus latency < 3 seconds target
- **Bug Fixes**: Fix any defects discovered during testing

# Implementation Plan

## Step 1: Single Consensus Round Tests
- Complete prepare → precommit → commit flow
- Block proposal and acceptance
- Vote collection and QC formation
- Block commitment and finality

## Step 2: Multi-Round Consensus Tests
- Sequential consensus rounds
- Chain growth and height advancement
- Leader rotation between rounds
- State consistency across rounds

## Step 3: 5-Node Network Tests
- Network initialization and setup
- All nodes participating in consensus
- Leader rotation through all nodes
- Consistent block commitment across nodes

## Step 4: State Synchronization Tests
- Node state consistency validation
- Committed chain synchronization
- View number synchronization
- QC propagation and consistency

## Step 5: Performance and Timing Tests
- Consensus round latency measurement
- Block proposal to commitment timing
- Network message propagation delays
- Performance baseline establishment

## Step 6: Edge Cases in Normal Operation
- Genesis block handling
- Empty payload blocks
- Rapid successive proposals
- View timeout and recovery

# Verification and Validation

## Architecture integrity
- Complete system integration working correctly
- All components collaborating properly

## Security
- Safety properties maintained in multi-node scenarios
- Liveness properties ensuring progress

## Performance
- Consensus latency meets <3 second target
- Efficient multi-node coordination

## Reliability
- Consistent state across all participating nodes
- Robust handling of normal operation edge cases

# Restrictions
- Focus on happy path scenarios only
- Achieve consensus latency <3 seconds
- Fix all discovered bugs before completion
- All tests must pass consistently across multiple runs