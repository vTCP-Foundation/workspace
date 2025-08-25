# 03-02 - Implement CLI commands for exchange rates

# Links
- [PRD](../../../prd/vtcpd-cli/03-exchange-rates-cli-and-http.md)
- [Related PRD in vtcpd](../../../prd/vtcpd/03-exchange-rates-manager.md)

# Description
Implement CLI commands in vtcpd-cli for managing exchange rates, mirroring vtcpd commands (SET:RATE, GET:RATE, LIST:RATES, DEL:RATE, CLEAR:RATES). Provide two input modes for setting a rate (real decimal and native value+shift), perform validations and conversions, and print results consistent with existing CLI outputs.

# Requirements and DOD
- Add new file `internal/handler/rates.go` implementing rates handling in the same style as other handler modules:
  - `HandleRates()` top-level entry (dispatched via `--type` flag): `set`, `set-native`, `get`, `list`, `del`, `clear`.
  - For `set`: flags `--from <equivalent_from>`, `--to <equivalent_to>`, `--real <real_rate>`; optional `--min <min_exchange_amount>`, `--max <max_exchange_amount>`.
  - For `set-native`: flags `--from`, `--to`, `--value <int>`, `--shift <int16>`; optional `--min`, `--max`.
  - For `get`: flags `--from`, `--to`.
  - For `list`: no flags.
  - For `del`: flags `--from`, `--to`.
  - For `clear`: no flags.
- Update dispatch in `internal/cmd_handler/cmd_handler.go` and `internal/cmd_handler/cmd_handler_testing.go` to route `rates` command to the handler (no testing HTTP endpoints required).
- Validations for set:
  - Exactly one mode used per subcommand invocation (enforced by subcommand split).
  - For `set`: `real_rate` must have ≤ 16 fractional digits.
  - For `set-native`: `shift` must be int16.
  - Decimals map must contain both equivalents: `{101:2, 1001:8, 1002:8, 2002:6}`.
- Conversion rules (shared with HTTP):
  - Real → native `(value, shift)` normalization; then `shift -= (decimals_from - decimals_to)`; validate int16.
  - For display in `get`/`list`, compute `real_rate` from native and scales, truncating toward zero for negative shifts.
- Outputs:
  - For `set`, `set-native`, `del`, `clear`: print empty `data` structure consistent with project style (no additional payload).
  - For `get`, `list`: print fields: `equivalent_from`, `equivalent_to`, `value`, `shift`, `real_rate`, `min_exchange_amount`, `max_exchange_amount`, `expires_at_unix_microseconds`.
- No regressions to existing commands. Build and linter pass.

# Implementation Plan
- Introduce `internal/handler/rates.go` with structure mirroring `transactions.go` or `settlement_lines.go` patterns (flag parsing, validation, node interaction, output formatting).
- Implement helper functions:
  - `parseAndValidateRealRate(real string) -> (value string, shift int)`
  - `applyScaleAdjustment(shift int, fromEq, toEq string) -> (int, error)` using constants map
  - `computeRealRateString(value string, shift int, fromEq, toEq string) -> string`
- Update `cmd_handler.go` and `cmd_handler_testing.go` `HandleCommand` switches to support `"rates"` and call into the node handler.
- Ensure node communication requests for rates follow existing FIFO protocol and formats (commands, tokens separators) used in the project.

# Test Plan
- Manual CLI invocations:
  - `vtcpd-cli rates --type set --from 1001 --to 2002 --real 0.123` → success, empty data; verify via `get`.
  - `vtcpd-cli rates --type set-native --from 1001 --to 2002 --value 123456 --shift 3` → success; verify via `get`.
  - `vtcpd-cli rates --type get --from 1001 --to 2002` → shows both native and real, plus min/max and expiry.
  - `vtcpd-cli rates --type list` → shows collection.
  - `vtcpd-cli rates --type del --from 1001 --to 2002` → success, empty data; verify removed.
  - `vtcpd-cli rates --type clear` → success, empty data; verify list empty.
  - Negative cases: real more than 16 decimals; missing scales; shift overflow.

# Verification and Validation
- Architect reviews CLI UX, outputs, error messages against PRD and existing style.

## Architecture integrity
- Handlers and command dispatch align with existing module structure; shared response structs used where applicable.

## Security
- Validate inputs; avoid shelling out; handle unexpected inputs gracefully.

## Performance
- Minimal overhead; O(1) conversions.

## Scalability
- Stateless; consistent with other handlers.

## Reliability
- Clear validation and error messages; no panics.

## Maintainability
- Readable code with small helpers and constants; aligned with code style.

## Cost
- No external dependencies.

## Compliance
- Adheres to policy and PRD; printing and formatting match project conventions.

# Restrictions
- Commit changes only after successfully executing the CLI demos described above.
