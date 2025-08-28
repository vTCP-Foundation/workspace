# 04-01 – EquivalentsSubsystemsRouter migration (unified IDs)

# Links
- [PRD](../../../prd/vtcpd/04-exchange-flow-calculation.md)

# Description
Migrate participant identification from `TopologyTrustLinesManager` to `EquivalentsSubsystemsRouter` to ensure a single, unified `ContractorID` space across all equivalents. Update all code that relied on local, per-equivalent ID tables.

# Requirements and DOD
- `EquivalentsSubsystemsRouter` provides unified assignment/resolution of `ContractorID` for any `BaseAddress`.
- Remove local ID tables from `TopologyTrustLinesManager` (former `mParticipantsAddresses`, counters, etc.).
- All consumers switch to the public `EquivalentsSubsystemsRouter` API.
- Legacy single-equivalent behavior remains unchanged (no regressions).
- Unit tests cover uniqueness, stability across equivalents, and repeated lookups.
- Build and run tests in `build-tests`.

# Implementation Plan
1. Add/clarify public API in `EquivalentsSubsystemsRouter`:
   - `ContractorID getOrCreateParticipantID(const BaseAddress&)`.
   - `optional<ContractorID> resolveParticipantID(const BaseAddress&)`.
   - Internal structures mapping addresses → IDs with atomic ID generation.
2. Remove any local participant tables and dependent code from `TopologyTrustLinesManager`.
3. Replace former usages across topology modules with `EquivalentsSubsystemsRouter` calls.
4. Use `mEquivalentsSubsystemsRouter` to access unified ID management where needed.
5. Refactor imports/dependency injection accordingly.

# Test Plan
- Unit tests (no integration):
  - `getOrCreateParticipantID` returns the same ID for the same address across equivalents.
  - `resolveParticipantID` correctly distinguishes known/unknown addresses.
  - Concurrent calls (if applicable) must not produce duplicates.
- Build and run in `build-tests`.

# Verification and Validation
Architect verifies all code paths no longer rely on local IDs in `TopologyTrustLinesManager` and that ID unification matches the PRD.

## Architecture integrity
Single source of truth for `ContractorID` in `EquivalentsSubsystemsRouter` as per PRD; eliminates duplication.

## Security
No changes to security posture or channels.

## Performance
Lookup/registration remains O(log N) or better.

## Scalability
Supports hundreds of thousands of participants without degradation.

## Reliability
ID stability guaranteed during process lifetime.

## Maintainability
Clean public API, minimal dependencies, solid unit test coverage.

## Cost
No additional infrastructure costs.

## Compliance
Complies with PRD and repository policy.

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)

