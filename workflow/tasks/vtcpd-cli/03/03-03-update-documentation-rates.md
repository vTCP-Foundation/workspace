# 03-03 - Update documentation for exchange rates (HTTP + CLI)

# Links
- [PRD](../../../prd/vtcpd-cli/03-exchange-rates-cli-and-http.md)
- [Related PRD in vtcpd](../../../prd/vtcpd/03-exchange-rates-manager.md)

# Description
Update `readme.md` to document new HTTP endpoints and CLI commands for exchange rates. Add explanation of native storage format (value + shift), reasons (cross-platform stability), conversion rules, precision limits, and examples.

# Requirements and DOD
- Update `readme.md`:
  - Add a new section in REST API describing:
    - POST `/api/v1/node/rates/{equivalent_from}/{equivalent_to}/` with query params and mutual exclusivity validation; response `{ "data": {} }`
    - GET `/api/v1/node/rates/{equivalent_from}/{equivalent_to}/` returning full rate object
    - GET `/api/v1/node/rates/` returning list and count
    - DELETE `/api/v1/node/rates/{equivalent_from}/{equivalent_to}/` with empty data
    - DELETE `/api/v1/node/rates/` with empty data
  - Add a new subsection in CLI documenting `rates --type` commands:
    - `set`, `set-native`, `get`, `list`, `del`, `clear` with flags and examples
    - Outputs: empty data for mutating commands; full object(s) for get/list
  - Add a background subsection explaining storage as `(value, shift)`, shift semantics, decimals map influence, and that real inputs are converted before sending to node; mention 16-decimal limit and int16 shift; note truncation behavior for negative shifts and potential precision impact.
- Keep style, tone, and formatting consistent with existing sections.
- Include example curl and CLI commands following current documentation style.

# Implementation Plan
- Mirror the structure used in Channels/Settlement Lines docs within `readme.md`.
- Provide JSON examples matching response shapes used in code.
- Add notes where validation applies (mutual exclusivity; decimals ≤ 16; int16 shift; scales map keys required: 101→2, 1001→8, 1002→8, 2002→6).

# Test Plan
- Documentation review: ensure all endpoints and commands are correctly represented with accurate paths, params, and examples.
- Validate examples by manual runs after implementation (post-merge sanity check).

# Verification and Validation
- Architect approval of documentation clarity and completeness.

## Architecture integrity
- Documentation reflects actual implemented interfaces and shares terminology with code.

## Security
- Document validation that prevents malformed inputs.

## Performance
- N/A for docs.

## Scalability
- N/A for docs.

## Reliability
- Examples are reproducible; descriptions unambiguous.

## Maintainability
- Sections placed consistently for easy future updates.

## Cost
- N/A.

## Compliance
- Aligns with policy and PRD; uses existing README style.

# Restrictions
- Commit changes only after verifying that examples reflect the implemented behavior.
