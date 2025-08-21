# 03-02 - Exchange rates commands, transactions, and wiring

# Links
- [PRD](../../../prd/vtcpd/03-exchange-rates-manager.md)

# Description
Implement commands and transactions for managing exchange rates and wire them into the transactions system, including result serialization via `ResultsInterface` and integration into `Core`/`TransactionsManager`.

# Requirements and DOD
- **Commands (parsing/identifiers/serialization)** — directory `src/core/interface/commands_interface/commands/rates/`:
  - Identifiers: `SET:RATE`, `GET:RATE`, `LIST:RATES`, `DEL:RATE`, `CLEAR:RATES`.
  - Payload formats as per PRD.
  - Follow existing parsing style and token formats (`kTokensSeparator`, `kCommandsSeparator`).
- **Transactions** — directory `src/core/transactions/transactions/rates/`:
  - `SetExchangeRateTransaction`: validate inputs, call `ExchangeRatesManager::addOrUpdate`, return `transactionUUID` + status code.
  - `GetExchangeRateTransaction`: fetch a single rate and serialize the tuple to `ResultsInterface`.
  - `ListExchangeRatesTransaction`: list all rates, serialize count and repeated tuples.
  - `RemoveExchangeRateTransaction`: remove a pair, return `transactionUUID` + status code.
  - `ClearExchangeRatesTransaction`: clear all rates, return `transactionUUID` + status code.
  - Single tuple format: `equivalentFrom, equivalentTo, exchangeRate, exchangeRateShift, minExchangeAmount, maxExchangeAmount, expiresAtUnixMicroseconds`.
- **Integration**:
  - Extend `TransactionsManager` (constructor and `processCommand(...)`) to support new commands/transactions; inject `ExchangeRatesManager *`.
  - Initialize `ExchangeRatesManager` in `Core` and pass it to `TransactionsManager` (same pattern as other dependencies).
- **Logging**: add debug logs for transaction execution and parsing/execution errors.
- **Definition of Done**:
  - Commands and transactions compile and run locally (smoke scenarios).
  - Result serialization matches the PRD formats.
  - Initialization/wiring in `Core` and `TransactionsManager` is complete with no regressions.
  - All existing project unit tests pass in `build-tests`.

# Implementation Plan
1. Add a new commands subdirectory `src/core/interface/commands_interface/commands/rates/` with corresponding parsing command classes.
2. Add transactions in `src/core/transactions/transactions/rates/` with behavior per PRD.
3. Extend `TransactionsManager`:
   - Add constructor parameter: `ExchangeRatesManager *` and store it as a member.
   - Handle new identifiers in `processCommand(...)` by instantiating the respective transactions.
4. Update `Core`:
   - Create `unique_ptr<ExchangeRatesManager>` with `mIOCtx` and `*mLog`.
   - Pass it to `TransactionsManager`.
5. Serialize via `ResultsInterface` for `GET`/`LIST` in the defined format; for `SET`/`DEL`/`CLEAR` return only `transactionUUID` and status code.
6. Review existing serialization examples (trust lines/history) and align token formatting.

# Test Plan
- Focus on smoke testing of commands and transactions within the existing unit-test infrastructure:
  - Prepare cases with valid/invalid payloads to verify parsing/serialization without external IO.
  - Verify `TransactionsManager::processCommand(...)` routes new identifiers correctly.
- Run tests only in `build-tests`:
  - `mkdir -p build-tests && cd build-tests`
  - `cmake .. -DCMAKE_BUILD_TYPE=Debug`
  - `make -j`
  - `ctest -V`

# Verification and Validation
## Architecture integrity
- Commands/transactions implemented within the existing structure; dependencies are injected via constructors; no side effects on other subsystems.

## Security
- Inputs are validated; no external IO; errors surfaced via standard statuses/exceptions.

## Performance
- Adding new command handling does not impact critical paths; operations are lightweight.

## Scalability
- Easy to extend with additional commands in the future; tuple format is stable.

## Reliability
- Unified serialization; errors are handled and logged properly.

## Maintainability
- Follows existing patterns for `ResultsInterface`, `TransactionsManager`, and command classes; responsibilities are clearly separated.

## Cost
- Minimal; no new external dependencies.

## Compliance
- Conforms to the PRD and project policies; unit tests only, no Docker.

# Restrictions
- make commits only after successful local execution and passing all tests in `build-tests`, and after verifying serialization formats per the PRD.

