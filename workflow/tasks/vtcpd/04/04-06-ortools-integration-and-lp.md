# 04-06 – OR-Tools integration and LP-based flow/paths computation

# Links
- [PRD](../../../prd/vtcpd/04-exchange-flow-calculation.md)

# Description
Integrate OR-Tools (LinearSolver), build an LP model per PRD with a path-based approach, compute maximum receivable amount in `mEquivalent`, and persist detailed path information.

# Requirements and DOD
- CMake: `find_package(ortools REQUIRED)`, link `ortools::ortools`.
- Node startup verification: check OR-Tools availability and minimum version (>= 9.0), terminate with error if unavailable or insufficient version.
- LP solver: create `MPSolver` (GLOP or equivalent), path variables, objective, constraints per PRD.
- Implement stages: path enumeration, LP build, optimization, results stored in `mMaxFlows` and `mOptimalPathResults`.
- Path logs in the new specified format: F(nodes: [addr1 -> addr2]; ids [id1 -> id2]; flow: amount; eq: equivalent) and E(node: addr; id: nodeId; eqs [fromEq->toEq]; flows:[inputFlow->outputFlow]).
- Runtime error handling for individual calculation failures.
- Unit tests for components (enumeration, model build, result interpretation) with mocks.
- Build and run tests in `build-tests`.

# Implementation Plan
1. Integrate OR-Tools in CMake, add `WITH_ORTOOLS` define.
2. Add startup verification logic to check OR-Tools availability and version.
3. Implement path enumeration (length cap, pruning non-viable paths).
4. Build LP: variables per path, objective coefficients = effective rates.
5. Add base constraints (sender balance, non-negativity, path upper bounds).
6. Optimize, collect results, log paths in new format.
7. Handle solver statuses/errors and startup failures.

# Test Plan
- Unit tests on small synthetic graphs:
  - Single path, single exchange – validate result.
  - Multiple alternative paths – pick the best.
  - No paths – zero result.
  - Path logging format validation for F and E elements.
  - Startup verification tests (mock OR-Tools availability/version checks).
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


