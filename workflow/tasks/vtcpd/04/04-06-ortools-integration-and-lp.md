# 04-06 – OR-Tools integration and LP-based flow/paths computation

# Links
- [PRD](../../../prd/vtcpd/04-exchange-flow-calculation.md)

# Description
Integrate OR-Tools (LinearSolver), build an LP model per PRD with a path-based approach, compute maximum receivable amount in `mEquivalent`, and persist detailed path information.

# Requirements and DOD
- CMake: `find_package(ortools REQUIRED)`, link `ortools::ortools`.
- LP solver: create `MPSolver` (GLOP or equivalent), path variables, objective, constraints per PRD.
- Implement stages: path enumeration, LP build, optimization, results stored in `mMaxFlows` and `mOptimalPathResults`.
- Path logs in the specified format.
- Graceful error if OR-Tools is unavailable.
- Unit tests for components (enumeration, model build, result interpretation) with mocks.
- Build and run tests in `build-tests`.

# Implementation Plan
1. Integrate OR-Tools in CMake, add `WITH_ORTOOLS` define.
2. Implement path enumeration (length cap, pruning non-viable paths).
3. Build LP: variables per path, objective coefficients = effective rates.
4. Add base constraints (sender balance, non-negativity, path upper bounds).
5. Optimize, collect results, log paths.
6. Handle solver statuses/errors.

# Test Plan
- Unit tests on small synthetic graphs:
  - Single path, single exchange – validate result.
  - Multiple alternative paths – pick the best.
  - No paths – zero result.
- Execute in `build-tests`.

# Verification and Validation
Architect validates model correctness and results against PRD baselines.

## Architecture integrity
LP computation isolated within the transaction; protocols unaffected.

## Security
No changes.

## Performance
Within NFR for typical network sizes.

## Scalability
Length/paths-count limits to control complexity.

## Reliability
Proper handling of INFEASIBLE/ABNORMAL statuses.

## Maintainability
Structured code, explicit interfaces, tests.

## Cost
No additional costs.

## Compliance
Complies with PRD and repository policy.

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)


