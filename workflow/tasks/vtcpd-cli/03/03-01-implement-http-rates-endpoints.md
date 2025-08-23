# 03-01 - Implement HTTP endpoints for exchange rates

# Links
- [PRD](../../../prd/vtcpd-cli/03-exchange-rates-cli-and-http.md)
- [Related PRD in vtcpd](../../../prd/vtcpd/03-exchange-rates-manager.md)

# Description
Implement HTTP endpoints in vtcpd-cli to operate exchange rates introduced in vtcpd PRD 03. Endpoints must follow existing router/handler style, perform validations, convert real decimal to native (value, shift), and return responses consistent with the project conventions.

# Requirements and DOD
- Add new file `internal/routes/rates.go` with handlers:
  - `SetRate(w http.ResponseWriter, r *http.Request)`
  - `GetRate(w http.ResponseWriter, r *http.Request)`
  - `ListRates(w http.ResponseWriter, r *http.Request)`
  - `DeleteRate(w http.ResponseWriter, r *http.Request)`
  - `ClearRates(w http.ResponseWriter, r *http.Request)`
- Register endpoints in `internal/server/server.go`:
  - `POST /api/v1/node/rates/{equivalent_from}/{equivalent_to}/` → `SetRate`
  - `GET /api/v1/node/rates/{equivalent_from}/{equivalent_to}/` → `GetRate`
  - `GET /api/v1/node/rates/` → `ListRates`
  - `DELETE /api/v1/node/rates/{equivalent_from}/{equivalent_to}/` → `DeleteRate`
  - `DELETE /api/v1/node/rates/` → `ClearRates`
- Validations for Set:
  - Exactly one mode required: `real_rate` XOR (`value` AND `shift`). If all three present or neither/missing pair → 400
  - `real_rate` fractional part length ≤ 12, else 400
  - `shift` must be int16, else 400
  - Decimals map must contain both `equivalent_from` and `equivalent_to`: `{101:2, 1001:8, 1002:8, 2002:6}`, else 400
- Conversion rules for Set:
  - `real_rate` → normalize to `(value, shift)` (e.g., 0.123→123,-3; 0.00123→123,-5; 123.456→123456,3; 1234→1234,0)
  - Apply scale difference: `shift -= (decimals_from - decimals_to)`; then validate int16
- Responses:
  - Set/Delete/Clear: JSON with `{ "data": {} }` (empty data)
  - Get: `data.rate` includes `equivalent_from`, `equivalent_to`, `value`, `shift`, `real_rate`, `min_exchange_amount`, `max_exchange_amount`, `expires_at_unix_microseconds`
  - List: `data.count`, `data.rates[]` with the same fields as Get
- Add response structs to `internal/common/common.go`:
  - `RatesActionResponse` (empty data)
  - `RateItem`, `RateGetResponse`, `RatesListResponse`
- Error handling and logging consistent with existing handlers; use 400 for validation errors.
- Build passes, linter passes.

# Implementation Plan
- Create `internal/routes/rates.go` following style of existing modules (e.g., `settlement_lines.go`).
- Implement parsing of path vars `equivalent_from`, `equivalent_to` and query vars.
- Implement validation helpers for mode selection, decimal length, int16 range.
- Implement conversion `real_rate`→`(value, shift)` and scale adjustment with constants map.
- Implement inverse conversion to compute `real_rate` string for Get/List outputs based on node-returned `(value, shift)` and decimals map (truncate toward zero for negative shifts).
- Integrate into router in `internal/server/server.go` (no changes to testing server).
- Add response model types in `internal/common/common.go` and reuse existing write/serialization patterns.

# Test Plan
- Manual curl checks against a node with exchange rates feature:
  - POST set with `real_rate=0.123` and `from=1001`, `to=2002`; expect 200 and empty data; verify in GET.
  - POST set-native with `value=123456`, `shift=3`; expect 200 and empty data; verify in GET.
  - GET one: verify all fields present; `real_rate` matches computed presentation.
  - GET list: verify count and array contents.
  - DELETE one: expect 200 and empty data; verify GET returns NotFound style (mapped to count=0 or 404 per existing conventions).
  - DELETE all: expect 200 and empty data; verify GET list empty.
  - Validation negatives: both modes present → 400; more than 12 decimals → 400; missing scale → 400; out-of-range shift → 400.

# Verification and Validation
- Architect reviews endpoints, validations, and responses against PRD.

## Architecture integrity
- Endpoints registered in `server.go`; handlers live in `internal/routes/`; response types centralized in `internal/common/common.go`.

## Security
- Validate and sanitize query/path inputs; no shelling out; standard HTTP status codes.

## Performance
- O(1) conversion; no heavy loops; list throughput depends on node response size only.

## Scalability
- Stateless handlers; no global mutable state beyond constants map.

## Reliability
- Clear validation errors; logs for failures; no panics.

## Maintainability
- Code follows existing patterns; constants extracted; helper functions small and testable.

## Cost
- No external services; negligible runtime cost.

## Compliance
- Follows project policy and PRD; response formatting consistent with existing APIs.

# Restrictions
- Commit changes only after successfully executing the demo (manual curl tests) or after successfully passing any added automated checks.
