# 14-01 - RPC Core Infrastructure

# Links
- [PRD 14: Async Observer RPC Communication](../../../prd/vtcpd/14-async-observer-rpc-communication.md)

# Description
Створити базову інфраструктуру для async RPC: enums `RpcMethod`, `RpcResponseStatus`, базові класи `RpcRequest` і `RpcResponse`, а також окремий CMake-таргет `network__rpc` в `src/core/network/rpc`. Це фундамент для подальших async RPC компонентів.

# Requirements and DOD
- Додати `RpcMethod` enum з методами Unknown, GetBlockNumber, AcceptClaim, GetClaimStatus, SubmitClaimVotes, GetClaimStatuses.
- Додати `RpcResponseStatus` enum з Success, Timeout, NetworkError, ParseError, RpcError.
- Додати базовий `RpcRequest` із збереженням `TransactionUUID`, pure virtual `method()`, typedef `Shared`.
- Додати базовий `RpcResponse` із збереженням `TransactionUUID`, `RpcResponseStatus`, `errorMessage`, `isSuccess()`, pure virtual `method()`, typedef `Shared`.
- Створити файлову структуру `src/core/network/rpc/` згідно PRD.
- Додати новий CMake таргет `network__rpc`, що лінкується з common, logger, contractors, observing, Boost::asio, nlohmann_json; підключити в головний CMake та unit_tests.
- Код компілюється без попереджень.

# Implementation Plan
- Створити каталоги `src/core/network/rpc`, `requests`, `responses`.
- Реалізувати `RpcMethod.h`, `RpcResponseStatus.h`, `RpcRequest.h/.cpp`, `RpcResponse.h/.cpp` з потрібними полями/методами.
- Додати `CMakeLists.txt` для `network__rpc`; підключити в кореневий CMake та unit_tests.
- Переконатися у відсутності залежностей на інші нові класи (мінімальний набір).

# Test Plan
- Побудувати проєкт (CMake) і переконатися у відсутності помилок/попереджень.
- Юніт-тести будуть у окремих задачах (див. тестові задачі цього PRD).

# Verification and Validation
<Architect approval that validation criteria appropriate to task complexity have been met>

## Architecture integrity
- [ ] Базові класи і enum-и розміщені в `src/core/network/rpc`, відповідають стилю існуючих мережевих модулів.
- [ ] CMake інтеграція не порушує інші таргети.

## Security
- [ ] Немає сторонніх побічних ефектів, лише декларативні типи.

## Performance
- [ ] Без накладних витрат (оголошення/мінімальна логіка).

## Scalability
- [ ] Базові типи дозволяють додавати нові RPC методи без змін існуючого коду.

## Reliability
- [ ] Базові класи містять мінімальні поля для ідентифікації транзакцій і статусів.

## Maintainability
- [ ] Коментарі/іменування відповідають наявним код-стандартам.
- [ ] Структура файлів відповідає схемі в PRD.

## Cost
- [ ] Немає додаткових залежностей поза вказаними в PRD.

## Compliance
- [ ] Відповідає PRD та проектним політикам.

# Restrictions
- Commit changes тільки після успішної побудови (демо/тести для цієї задачі не передбачені). 
