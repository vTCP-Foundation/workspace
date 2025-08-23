## [03] vtcpd-cli Exchange Rates: CLI and HTTP Integration

## Document Information
- **Project Name**: vtcpd-cli
- **PRD ID**: 03
- **Phase/Iteration**: PRD 03
- **Document Version**: 1.0
- **Date**: 2025-08-23
- **Author(s)**: Mykola Ilashchuk, Dima Chizhevsky
- **Stakeholders**: Dima Chizhevsky, Mykola Ilashchuk
- **PRD Status**: 1.1 - PRD file created
- **Last Status Update**: 2025-08-23
- **Previous PRD**: workspace/workflow/prd/vtcpd/03-exchange-rates-manager.md
- **Related Documents**: 
  - vtcpd tasks: workspace/workflow/tasks/vtcpd/03/
  - Policy: workspace/policy.md

## Executive Summary
Add HTTP endpoints and CLI commands in vtcpd-cli to operate exchange rates newly introduced in vtcpd (PRD 03). The CLI supports two input modes for setting a rate: native (value + shift) and real decimal. Real decimal is converted to native (value, shift) with validation and scale adjustments. Read endpoints return both native and computed real representation. Documentation is updated accordingly.

## Iteration Context
### Previous Iterations Summary
- vtcpd implemented in-memory ExchangeRatesManager and commands (SET:RATE, GET:RATE, LIST:RATES, DEL:RATE, CLEAR:RATES) outside main (feature branch prd/03-exchange-rates-manager).

### Current State Analysis
- vtcpd-cli has established HTTP routing and CLI handlers for channels, settlement lines, payments, etc., but lacks exchange-rate integration.

## Problem Statement
### Background
vtcpd stores exchange rates in a platform-stable format as a pair of integers: value and base-10 shift.

### Problem Description
- vtcpd-cli needs HTTP and CLI surfaces to send the newly added vtcpd rate commands and present results to users.
- Real-rate inputs must be converted to native representation with scale corrections and validations.

### Success Metrics
- Implemented endpoints and CLI commands operate successfully against a node with the feature branch.
- Validation rejects malformed inputs (e.g., triple-parameter set, excessive decimals, missing scales, out-of-range shift).
- Documentation covers usage, formats, and precision caveats.

## Project Scope
### This Iteration's Scope
#### New Features/Enhancements
- HTTP endpoints under `server.go` for: set, get, list, delete, clear exchange rates.
- CLI command group `rates --type <...>` with set (real), set-native, get, list, del, clear.
- Decimal-to-native conversion with scale adjustment and validations.
- Response structures centralized in `internal/common/common.go`.
- Documentation updates in `readme.md`.

#### Bug Fixes & Technical Improvements
- None beyond new features.

#### Modifications to Existing Features
- Extend HTTP router (`internal/server/server.go`).
- Add new handlers modules: `internal/routes/rates.go`, `internal/handler/rates.go`.
- Update command dispatchers: `internal/cmd_handler/cmd_handler.go`, `internal/cmd_handler/cmd_handler_testing.go` (types and routing only; endpoints exist only in main server per requirement).

### Explicitly Out of Scope
- Changes inside vtcpd core (manager/commands/transactions) beyond interfacing.
- Testing server endpoints (`server_testing.go`).
- Persistence of rate scales in config (kept as code constants for now).

### Dependencies from Previous Iterations
- vtcpd PRD 03 exchange rates functionality (commands and result serialization conventions).

### Future Roadmap Impact
- Enables future exchange flows and UI tooling; future enhancement to load decimals map from config.

## User Stories & Requirements
### Functional Requirements
1. HTTP Endpoints (main server only):
   - POST `/api/v1/node/rates/{equivalent_from}/{equivalent_to}/`
     - Query parameters:
       - Either `real_rate` (string) XOR (`value` string AND `shift` int16)
       - Optional: `min_exchange_amount` (string), `max_exchange_amount` (string)
     - Rules:
       - If all three `real_rate`, `value`, `shift` provided → validation error
       - If none or only part provided → validation error
       - For `real_rate`: maximum 12 digits after decimal point; else validation error
       - `shift` must be within int16 range
       - Decimal scales map must contain both equivalents; else validation error
       - Convert `real_rate` to native (value, shift) and then apply scale adjustment: `shift -= (decimals_from - decimals_to)`
     - Response: `{ "data": {} }` (empty data)
   - GET `/api/v1/node/rates/{equivalent_from}/{equivalent_to}/`
     - Response `data.rate` fields: `equivalent_from`, `equivalent_to`, `value`, `shift`, `real_rate`, `min_exchange_amount`, `max_exchange_amount`, `expires_at_unix_microseconds`
   - GET `/api/v1/node/rates/`
     - Response `data.count`, `data.rates[]` with same fields as single get
   - DELETE `/api/v1/node/rates/{equivalent_from}/{equivalent_to}/`
     - Response: `{ "data": {} }`
   - DELETE `/api/v1/node/rates/`
     - Response: `{ "data": {} }`

2. CLI Commands: `vtcpd-cli rates --type <subcommand>`
   - `set`: flags `--from <equivalent_from>`, `--to <equivalent_to>`, `--real <real_rate>`; optional `--min <min_exchange_amount>`, `--max <max_exchange_amount>`
   - `set-native`: flags `--from`, `--to`, `--value <int>`, `--shift <int16>`; optional `--min`, `--max`
   - `get`: flags `--from`, `--to`
   - `list`: no flags
   - `del`: flags `--from`, `--to`
   - `clear`: no flags
   - Outputs:
     - For set / set-native / del / clear: empty data structure (no payload beyond standard status)
     - For get / list: include native (`value`, `shift`) and computed `real_rate` plus `min_exchange_amount`, `max_exchange_amount`, and `expires_at_unix_microseconds`

3. Conversion and Validation
   - Decimals map (constant within CLI): `{ 101: 2, 1001: 8, 1002: 8, 2002: 6 }`
   - Real to native:
     - Normalize decimal to integer `value` with `shift` such that e.g. `0.123 → (123, -3)`, `0.00123 → (123, -5)`, `123.456 → (123456, 3)`, `1234 → (1234, 0)`
     - Apply scale difference: `shift -= (decimals_from - decimals_to)`
     - Validate: `real_rate` has <= 12 fractional digits; `shift` fits int16 after adjustment
   - Native to real (for display): compute decimal string using `value`, `shift`, and scale map (inverse of the above), without rounding for negative shifts (truncate toward zero semantics as per vtcpd)

4. Response Structures
   - Centralize in `internal/common/common.go`:
     - `RatesActionResponse` (empty `data` for set/del/clear)
     - `RateItem` (fields as above)
     - `RateGetResponse { rate: RateItem }`
     - `RatesListResponse { count: int, rates: []RateItem }`

5. Documentation
   - Update `readme.md`:
     - Add HTTP endpoints (paths, params, examples, responses)
     - Add CLI usage for `rates`
     - Explain native storage (value + shift) for platform stability; two input modes; conversion rules and precision caveats (12-decimal limit; truncation effects)

### Non-Functional Requirements
- Security: input validation; no external IO beyond existing node communication
- Performance: O(1) per-request conversion; list operations proportional to node response size
- Reliability: strict validation prevents malformed commands; error messages aligned with existing style
- Maintainability: follows existing file/module patterns; shared response structs in common package

## Technical Specifications
### Architecture Evolution
- Add modules: `internal/routes/rates.go`, `internal/handler/rates.go`.
- Extend router in `internal/server/server.go` to mount 5 endpoints listed above.
- Extend command handlers in `internal/cmd_handler/cmd_handler.go` and testing variant to dispatch `rates` commands (testing server endpoints not added).
- Add response structs to `internal/common/common.go`.

### HTTP Endpoints
- Base path: `/api/v1/node/rates/`
- Methods and behaviors strictly as defined in Functional Requirements.
- Error handling style mirrors existing endpoints (HTTP 400 for validation errors; 500 on internal issues; 200 with empty `data` on success for set/del/clear).

### CLI Integration
- New top-level command: `rates` with `--type` subcommands.
- Reuse existing CLI argument parsing and node communication pipeline.

### Conversion Utilities
- Helper functions in `internal/handler/rates.go`:
  - `parseAndValidateRealRate(...) -> (value string, shift int16)`
  - `computeRealRateString(value string, shift int16, eqFrom, eqTo string) -> string`
  - Use constants for decimals map; error if eq not present.

## Integration Requirements
- Wire routes into `server.go` in the same style as other groups (channels, settlement lines, transactions).
- Use `ResultsInterface`-compatible formatting conventions already present in the project for consistency.

## Test Plan (Proportional)
- Simple manual tests and basic validations (since this PRD is implementation-focused):
  - Validation: reject when both `real_rate` and `value+shift` present or missing.
  - Validation: `real_rate` fractional part length > 12 rejected.
  - Validation: missing decimals map entry rejected.
  - Conversion correctness: sample cases 0.123, 0.00123, 123.456, 1234 with different equivalent scales.
  - Readbacks: `real_rate` computed matches expected truncation behavior for negative shifts.

## Quality Gates
- Build succeeds; linter passes.
- Endpoints respond per spec; CLI commands operate per spec.
- `readme.md` updated with new endpoints and CLI usage.

## Deployment & Release Strategy
- Feature is additive; safe to release. No configuration migration.

## Success Metrics & Monitoring
- Input validation error rate measurable via logs; aim for clear messages.
- Correctness spot-checked against vtcpd responses on feature branch.

## Appendices
### Decimals Map
Fixed constants for equivalent fractional digits: `101 → 2`, `1001 → 8`, `1002 → 8`, `2002 → 6`.

### Error Conditions (Examples)
- 400: "real_rate and (value, shift) are mutually exclusive"
- 400: "real_rate has more than 12 fractional digits"
- 400: "shift is out of int16 range"
- 400: "unknown equivalent scale for {equivalent}"


